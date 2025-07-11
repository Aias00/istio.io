---
title: Helm を使ったインストール（簡易）
description: 単一のチャートで Helm Ambient モード対応の Istio をインストールします。
weight: 4
owner: istio/wg-environments-maintainers
test: yes
draft: true
---

{{< tip >}}
このガイドに従って、Ambient モード対応の Istio メッシュをインストールおよび構成します。Istio を初めて使用する場合や、試してみたいだけの場合は、[クイックスタートガイド](/ja/docs/ambient/getting-started)に従ってください。
{{< /tip >}}

本番環境での利用には、Helm を使った Ambient モードでの Istio インストールを推奨します。
制御されたアップグレードを可能にするため、コントロールプレーンとデータプレーンのコンポーネントは分割されてパッケージ化・インストールされます。
（Ambient データプレーンは[2 つのコンポーネント](/ja/docs/ambient/architecture/data-plane)、ztunnel と waypoint に分かれているため、アップグレード時はこれらのコンポーネントごとに個別の手順が必要です。）

## 前提条件 {#prerequisites}

1. [プラットフォーム固有の前提条件](/ja/docs/ambient/install/platform-prerequisites)を確認してください。

1. [Helm クライアント](https://helm.sh/docs/intro/install/)（バージョン 3.6 以上）をインストールしてください。

1. Helm リポジトリを設定します：

   {{< text syntax=bash snip_id=configure_helm >}}
   $ helm repo add istio https://istio-release.storage.googleapis.com/charts
   $ helm repo update
   {{< /text >}}

<!-- ### Base components -->

<!-- The `base` chart contains the basic CRDs and cluster roles required to set up Istio. -->
<!-- This should be installed prior to any other Istio component. -->

<!-- {{< text syntax=bash snip_id=install_base >}} -->
<!-- $ helm install istio-base istio/base -n istio-system --create-namespace --wait -->
<!-- {{< /text >}} -->

### Kubernetes Gateway API CRD のインストールまたはアップグレード {#install-or-upgrade-the-kubernetes-gateway-api-crds}

{{< boilerplate gateway-api-install-crds >}}

### Istio Ambient コントロールプレーンとデータプレーンのインストール {#install-the-istio-ambient-control-plane-and-data-plane}

`ambient` チャートは、Ambient に必要なすべての Istio データプレーンおよびコントロールプレーンコンポーネントをインストールします。
これは、各コンポーネントチャートをまとめた Helm ラッパーチャートです。

{{< warning >}}
すべてをこのラッパーチャートの一部としてインストールした場合、アップグレードやアンインストールもこのラッパーチャート経由でのみ可能です。サブコンポーネント単体でのアップグレードやアンインストールはできませんのでご注意ください。
{{< /warning >}}

{{< text syntax=bash snip_id=install_ambient_aio >}}
$ helm install istio-ambient istio/ambient --namespace istio-system --create-namespace --wait
{{< /text >}}

### イングレスゲートウェイ（オプション） {#ingress-gateway-optional}

{{< tip >}}
{{< boilerplate gateway-api-future >}}
Gateway API を利用している場合、以下の Helm チャートによるイングレスゲートウェイのインストール・管理は不要です。
詳細は [Gateway API タスク](/ja/docs/tasks/traffic-management/ingress/gateway-api/#automated-deployment) を参照してください。
{{< /tip >}}

イングレスゲートウェイをインストールするには、次のコマンドを実行します：

{{< text syntax=bash snip_id=install_ingress >}}
$ helm install istio-ingress istio/gateway -n istio-ingress --create-namespace --wait
{{< /text >}}

Kubernetes クラスターが `LoadBalancer` サービスタイプ（`type: LoadBalancer`）をサポートしておらず、正しい外部 IP が割り当てられない場合は、
無限に待機しないよう `--wait` オプションを外して上記コマンドを実行してください。
ゲートウェイのインストールに関する詳細は[ゲートウェイのインストール](/ja/docs/setup/additional-setup/gateway/)を参照してください。

## 設定 {#configuration}

Ambient ラッパーチャートは、以下のコンポーネント Helm チャートで構成されています：

- base
- istiod
- istio-cni
- ztunnel

1 つまたは複数の `--set <parameter>=<value>` パラメータでデフォルト値を変更できます。
または、`--values <file>` パラメータでカスタム値ファイルを指定して複数のパラメータを設定できます。

ラッパーチャート経由でも、個別インストール時と同様に、コンポーネント名を値パスの前に付けることで、
コンポーネント単位の設定上書きが可能です。

例：

{{< text syntax=bash snip_id=none >}}
$ helm install istiod istio/istiod --set hub=gcr.io/istio-testing
{{< /text >}}

は、ラッパーチャート経由では：

{{< text syntax=bash snip_id=none >}}
$ helm install istio-ambient istio/ambient --set istiod.hub=gcr.io/istio-testing
{{< /text >}}

のようになります。

各サブコンポーネントでサポートされている設定オプションやドキュメントを確認するには、次のコマンドを実行してください：

{{< text syntax=bash >}}
$ helm show values istio/istiod
{{< /text >}}

興味のある各コンポーネントについて実行できます。

Helm インストールの使い方やカスタマイズの詳細は、[サイドカーインストールドキュメント](/ja/docs/setup/install/helm/)を参照してください。

## インストールの検証 {#verify-the-installation}

### ワークロードの状態を検証する {#verify-the-workload-status}

すべてのコンポーネントをインストールした後、次のコマンドで Helm デプロイの状態を確認できます：

{{< text syntax=bash snip_id=show_components >}}
$ helm ls -n istio-system
NAME NAMESPACE REVISION UPDATED STATUS CHART APP VERSION
istio-ambient istio-system 1 2024-04-17 22:14:45.964722028 +0000 UTC deployed ambient-{{< istio_full_version >}} {{< istio_full_version >}}
{{< /text >}}

次のコマンドでデプロイ済み Pod の状態を確認できます：

{{< text syntax=bash snip_id=check_pods >}}
$ kubectl get pods -n istio-system
NAME READY STATUS RESTARTS AGE
istio-cni-node-g97z5 1/1 Running 0 10m
istiod-5f4c75464f-gskxf 1/1 Running 0 10m
ztunnel-c2z4s 1/1 Running 0 10m
{{< /text >}}

### サンプルアプリケーションでの検証 {#verify-with-the-sample-application}

Helm で Ambient モードをインストールした後、[サンプルアプリケーションのデプロイ](/ja/docs/ambient/getting-started/deploy-sample-app/)ガイドに従ってサンプルアプリとイングレスゲートウェイをデプロイできます。
その後、[アプリケーションをメッシュに追加](/ja/docs/ambient/getting-started/secure-and-visualize/#add-bookinfo-to-the-mesh)できます。

## アンインストール {#uninstall}

上記でインストールしたチャートをアンインストールすることで、Istio およびそのコンポーネントを削除できます。

1. すべての Istio コンポーネントをアンインストール

   {{< text syntax=bash snip_id=delete_ambient_aio >}}
   $ helm delete istio-ambient -n istio-system
   {{< /text >}}

1. （オプション）すべての Istio ゲートウェイチャートのインストールを削除：

   {{< text syntax=bash snip_id=delete_ingress >}}
   $ helm delete istio-ingress -n istio-ingress
   $ kubectl delete namespace istio-ingress
   {{< /text >}}

1. Istio がインストールした CRD を削除（オプション）

   {{< warning >}}
   これにより、作成されたすべての Istio リソースが削除されます。
   {{< /warning >}}

   {{< text syntax=bash snip_id=delete_crds >}}
   $ kubectl get crd -oname | grep --color=never 'istio.io' | xargs kubectl delete
   {{< /text >}}

1. `istio-system` 名前空間を削除：

   {{< text syntax=bash snip_id=delete_system_namespace >}}
   $ kubectl delete namespace istio-system
   {{< /text >}}
