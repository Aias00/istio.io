---
title: カナリアアップグレード
description: まずカナリアデプロイの新しいコントロールプレーンを実行して Istio をアップグレードします。
weight: 10
keywords: [kubernetes, upgrading, canary]
owner: istio/wg-environments-maintainers
test: yes
---

まずカナリアデプロイの新しいコントロールプレーンを実行して Istio をアップグレードします。これにより、すべてのトラフィックを新バージョンに移行する前に、一部のワークロードでアップグレードの影響を監視できます。
この方法は[インプレースアップグレード](/ja/docs/setup/upgrade/in-place/)よりも安全で、推奨されるアップグレード方法です。

Istio のインストール時、`revision` インストール設定を使うことで複数の独立したコントロールプレーンを同時にデプロイできます。アップグレード用のカナリアバージョンは、
異なる `revision` を使って旧バージョンの隣に新バージョンの Istio コントロールプレーンをインストール・起動できます。各リビジョンは独立した Istio コントロールプレーン（`Deployment`、`Service` など）を持ちます。

## アップグレード前 {#before-you-upgrade}

Istio をアップグレードする前に、`istioctl x precheck` コマンドを実行して、アップグレードが環境と互換性があるか確認することを推奨します。

{{< text bash >}}
$ istioctl x precheck
✔ No issues found when checking the cluster. Istio is safe to install or upgrade!
To get started, check out https://istio.io/latest/docs/setup/getting-started/
{{< /text >}}

{{< idea >}}
バージョンベースのアップグレードでは、2 つのマイナーバージョンをまたいだアップグレード（例：`1.15` から `1.17` へ）がサポートされています。
これはインプレースアップグレードとは異なり、インプレースの場合はすべての中間マイナーバージョンを順にアップグレードする必要があります。
{{< /idea >}}

## コントロールプレーン {#control-plane}

`canary` という新しいリビジョンをインストールするには、次のように `revision` フィールドを設定します：

{{< tip >}}
本番環境では、リビジョン名は Istio のバージョンに対応させるのが望ましいですが、
`revision={{< istio_full_version_revision >}}` のように `.` を `-` などに置き換えてください。
`.` は有効なリビジョン名の文字ではありません。
{{< /tip >}}

{{< text bash >}}
$ istioctl install --set revision=canary
{{< /text >}}

このコマンドを実行すると、2 つのコントロールプレーン Deployment と Service が並行して動作します：

{{< text bash >}}
$ kubectl get pods -n istio-system -l app=istiod
NAME READY STATUS RESTARTS AGE
istiod-{{< istio_previous_version_revision >}}-1-bdf5948d5-htddg 1/1 Running 0 47s
istiod-canary-84c8d4dcfb-skcfv 1/1 Running 0 25s
{{< /text >}}

{{< text bash >}}
$ kubectl get svc -n istio-system -l app=istiod
NAME TYPE CLUSTER-IP EXTERNAL-IP PORT(S) AGE
istiod-{{< istio_previous_version_revision >}}-1 ClusterIP 10.96.93.151 <none> 15010/TCP,15012/TCP,443/TCP,15014/TCP 109s
istiod-canary ClusterIP 10.104.186.250 <none> 15010/TCP,15012/TCP,443/TCP,15014/TCP 87s
{{< /text >}}

また、新バージョンを含む 2 つの Sidecar インジェクション設定も確認できます。

{{< text bash >}}
$ kubectl get mutatingwebhookconfigurations
NAME WEBHOOKS AGE
istio-sidecar-injector-{{< istio_previous_version_revision >}}-1 2 2m16s
istio-sidecar-injector-canary 2 114s
{{< /text >}}

## データプレーン {#data-plane}

[ゲートウェイのカナリアアップグレード](/ja/docs/setup/additional-setup/gateway/#canary-upgrade-advanced)も参照してください。
ここでは Istio Gateway の特定リビジョンのインスタンスを動作させる方法を説明しています。
この例では `default` Istio プロファイルを使っているため、Istio Gateway は特定リビジョンのインスタンスではなく、インプレースで新しいコントロールプレーンリビジョンにアップグレードされます。
`istio-ingress` Gateway が `canary` リビジョンを使っているかは、次のコマンドで確認できます：

{{< text bash >}}
$ istioctl proxy-status | grep "$(kubectl -n istio-system get pod -l app=istio-ingressgateway -o jsonpath='{.items..metadata.name}')" | awk -F '[[:space:]][[:space:]]+' '{print $8}'
istiod-canary-6956db645c-vwhsk
{{< /text >}}

ただし、新バージョンをインストールしただけでは既存の Sidecar プロキシには影響しません。アップグレードするには、
それらが新しい `istiod-canary` コントロールプレーンを参照するように設定する必要があります。
これは、ネームスペースラベル `istio.io/rev` を使った Sidecar インジェクションで制御します。

`test-ns` というネームスペースを作成し、`istio-injection` を有効化します。
そのネームスペースでサンプルの curl Pod をデプロイします：

1. `test-ns` ネームスペースを作成します。

   {{< text bash >}}
   $ kubectl create ns test-ns
   {{< /text >}}

1. `istio-injection` ラベルをネームスペースに付与します。

   {{< text bash >}}
   $ kubectl label namespace test-ns istio-injection=enabled
   {{< /text >}}

1. `test-ns` ネームスペースでサンプルの curl Pod を起動します。

   {{< text bash >}}
   $ kubectl apply -n test-ns -f samples/curl/curl.yaml
   {{< /text >}}

`test-ns` をアップグレードするには、`istio-injection` ラベルを削除し、`istio.io/rev` ラベルを `canary` リビジョンに設定します。
後方互換性のため、`istio-injection` ラベルは削除してください（`istio.io/rev` より優先されます）。

{{< text bash >}}
$ kubectl label namespace test-ns istio-injection- istio.io/rev=canary
{{< /text >}}

ネームスペースを更新したら、Pod を再起動して再インジェクションをトリガーする必要があります。
`test-ns` 内のすべての Pod を再起動するには：

{{< text bash >}}
$ kubectl rollout restart deployment -n test-ns
{{< /text >}}

Pod が再インジェクションされると、`istiod-canary` コントロールプレーンを参照するようになります。
`istioctl proxy-status` で確認できます。

{{< text bash >}}
$ istioctl proxy-status | grep "\.test-ns "
{{< /text >}}

出力には、そのネームスペースでリビジョンを使っているすべての Pod が表示されます。

## 安定リビジョンラベル {#stable-revision-labels}

{{< tip >}}
Helm を使っている場合は [Helm アップグレードガイド](/ja/docs/setup/upgrade/helm) を参照してください。
{{</ tip >}}

{{< boilerplate revision-tags-preamble >}}

### 使い方 {#usage}

{{< boilerplate revision-tags-usage >}}

1. 2 つのリビジョンのコントロールプレーンをインストールします：

   {{< text bash >}}
   $ istioctl install --revision={{< istio_previous_version_revision >}}-1 --set profile=minimal --skip-confirmation
   $ istioctl install --revision={{< istio_full_version_revision >}} --set profile=minimal --skip-confirmation
   {{< /text >}}

1. `stable` と `canary` のリビジョンラベルを作成し、それぞれのリビジョンに関連付けます：

   {{< text bash >}}
   $ istioctl tag set prod-stable --revision {{< istio_previous_version_revision >}}-1
   $ istioctl tag set prod-canary --revision {{< istio_full_version_revision >}}
   {{< /text >}}

1. アプリケーションのネームスペースにラベルを付与し、それぞれのリビジョンに関連付けます：

   {{< text bash >}}
   $ kubectl create ns app-ns-1
   $ kubectl label ns app-ns-1 istio.io/rev=prod-stable
   $ kubectl create ns app-ns-2
   $ kubectl label ns app-ns-2 istio.io/rev=prod-stable
   $ kubectl create ns app-ns-3
   $ kubectl label ns app-ns-3 istio.io/rev=prod-canary
   {{< /text >}}

1. 各ネームスペースでサンプルの curl Pod をデプロイします：

   {{< text bash >}}
   $ kubectl apply -n app-ns-1 -f samples/curl/curl.yaml
   $ kubectl apply -n app-ns-2 -f samples/curl/curl.yaml
   $ kubectl apply -n app-ns-3 -f samples/curl/curl.yaml
   {{< /text >}}

1. `istioctl proxy-status` コマンドでアプリケーションとコントロールプレーンのマッピングを確認します：

   {{< text bash >}}
   $ istioctl ps
   NAME CLUSTER CDS LDS EDS RDS ECDS ISTIOD VERSION
   curl-78ff5975c6-62pzf.app-ns-3 Kubernetes SYNCED SYNCED SYNCED SYNCED NOT SENT istiod-{{< istio_full_version_revision >}}-7f6fc6cfd6-s8zfg {{< istio_full_version >}}
   curl-78ff5975c6-8kxpl.app-ns-1 Kubernetes SYNCED SYNCED SYNCED SYNCED NOT SENT istiod-{{< istio_previous_version_revision >}}-1-bdf5948d5-n72r2 {{< istio_previous_version >}}.1
   curl-78ff5975c6-8q7m6.app-ns-2 Kubernetes SYNCED SYNCED SYNCED SYNCED NOT SENT istiod-{{< istio_previous_version_revision >}}-1-bdf5948d5-n72r2 {{< istio_previous_version_revision >}}.1
   {{< /text >}}

{{< boilerplate revision-tags-middle >}}

{{< text bash >}}
$ istioctl tag set prod-stable --revision {{< istio_full_version_revision >}} --overwrite
{{< /text >}}

{{< boilerplate revision-tags-prologue >}}

{{< text bash >}}
$ kubectl rollout restart deployment -n app-ns-1
$ kubectl rollout restart deployment -n app-ns-2
{{< /text >}}

`istioctl proxy-status` コマンドでアプリケーションとコントロールプレーンのマッピングを確認します：

{{< text bash >}}
$ istioctl ps
NAME CLUSTER CDS LDS EDS RDS ECDS ISTIOD VERSION
curl-5984f48bc7-kmj6x.app-ns-1 Kubernetes SYNCED SYNCED SYNCED SYNCED NOT SENT istiod-{{< istio_full_version_revision >}}-7f6fc6cfd6-jsktb {{< istio_full_version >}}
curl-78ff5975c6-jldk4.app-ns-3 Kubernetes SYNCED SYNCED SYNCED SYNCED NOT SENT istiod-{{< istio_full_version_revision >}}-7f6fc6cfd6-jsktb {{< istio_full_version >}}
curl-7cdd8dccb9-5bq5n.app-ns-2 Kubernetes SYNCED SYNCED SYNCED SYNCED NOT SENT istiod-{{< istio_full_version_revision >}}-7f6fc6cfd6-jsktb {{< istio_full_version >}}
{{< /text >}}

### デフォルトバージョン {#default-tag}

{{< boilerplate revision-tags-default-intro >}}

{{< text bash >}}
$ istioctl tag set default --revision {{< istio_full_version_revision >}}
{{< /text >}}

{{< boilerplate revision-tags-default-outro >}}

## 旧コントロールプレーンのアンインストール {#uninstall-old-control-plane}

コントロールプレーンとデータプレーンのアップグレード後、旧コントロールプレーンをアンインストールできます。
例えば、次のコマンドでリビジョン `{{< istio_previous_version_revision >}}-1` のコントロールプレーンをアンインストールします：

{{< text bash >}}
$ istioctl uninstall --revision {{< istio_previous_version_revision >}}-1 -y
{{< /text >}}

旧コントロールプレーンにリビジョンラベルがない場合は、元のインストールオプションでアンインストールしてください。例：

{{< text bash >}}
$ istioctl uninstall -f manifests/profiles/default.yaml -y
{{< /text >}}

旧コントロールプレーンが削除され、新しいコントロールプレーンだけがクラスタに存在することを確認します：

{{< text bash >}}
$ kubectl get pods -n istio-system -l app=istiod
NAME READY STATUS RESTARTS AGE
istiod-canary-55887f699c-t8bh8 1/1 Running 0 27m
{{< /text >}}

上記の手順は、指定したコントロールプレーンリビジョンのリソースのみを削除します。他のコントロールプレーンと共有されているクラスタスコープのリソースは削除されません。
Istio を完全にアンインストールするには、[アンインストールガイド](/ja/docs/setup/install/istioctl/#uninstall-istio) を参照してください。

## カナリアコントロールプレーンのアンインストール {#uninstall-canary-control-plane}

カナリアアップグレードを完了せずに旧コントロールプレーンにロールバックする場合は、次のコマンドでカナリアリビジョンをアンインストールできます：

{{< text bash >}}
$ istioctl uninstall --revision=canary -y
{{< /text >}}

ただし、この場合は旧バージョンの Gateway を手動で再インストールする必要があります。アンインストールコマンドはインプレースアップグレードされた Gateway を自動で復元しません。

{{< tip >}}
旧コントロールプレーンに対応する `istioctl` バージョンで旧 Gateway を再インストールし、
ダウンタイムを避けるため、旧 Gateway が起動・稼働してからカナリアアンインストールを実施してください。
{{< /tip >}}

## クリーンアップ {#cleanup}

1. 作成したリビジョンラベルをクリーンアップします：

   {{< text bash >}}
   $ istioctl tag remove prod-stable
   $ istioctl tag remove prod-canary
   {{< /text >}}

1. カナリアアップグレード用に作成したネームスペースとリビジョンラベルの例をクリーンアップします：

   {{< text bash >}}
   $ kubectl delete ns istio-system test-ns
   {{< /text >}}

1. カナリアアップグレード用に作成したネームスペースとリビジョンの例をクリーンアップします：

   {{< text bash >}}
   $ kubectl delete ns istio-system app-ns-1 app-ns-2 app-ns-3
   {{< /text >}}
