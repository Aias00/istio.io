---
title: Helm でのインストール
description: Helm を使って Ambient モード対応の Istio をインストールします。
weight: 4
owner: istio/wg-environments-maintainers
aliases:
  - /ja/docs/ops/ambient/install/helm-installation
  - /ja/latest/docs/ops/ambient/install/helm-installation
  - /ja/docs/ambient/install/helm-installation
  - /ja/latest/docs/ambient/install/helm-installation
test: yes
---

{{< tip >}}
このガイドに従って、Ambient モード対応の Istio メッシュをインストールおよび構成します。
Istio を初めて使用する場合や、試してみたいだけの場合は、[クイックスタートガイド](/ja/docs/ambient/getting-started)に従ってください。
{{< /tip >}}

本番環境での利用には、Helm を使った Ambient モードでの Istio インストールを推奨します。
制御されたアップグレードを可能にするため、コントロールプレーンとデータプレーンのコンポーネントは分割されてパッケージ化・インストールされます。
（Ambient データプレーンは ztunnel と waypoint の[2 つのコンポーネント](/ja/docs/ambient/architecture/data-plane)に分かれているため、これらのコンポーネントごとに個別のアップグレードが必要です。）

## 前提条件 {#prerequisites}

1. [プラットフォーム固有の前提条件](/ja/docs/ambient/install/platform-preventions)を確認してください。

1. [Helm クライアント](https://helm.sh/docs/intro/install/)（バージョン 3.6 以上）をインストールしてください。

1. Helm リポジトリを設定します：

   {{< text syntax=bash snip_id=configure_helm >}}
   $ helm repo add istio https://istio-release.storage.googleapis.com/charts
   $ helm repo update
   {{< /text >}}

### Kubernetes Gateway API CRD のインストールまたはアップグレード {#install-or-upgrade-the-kubernetes-gateway-api-crds}

{{< boilerplate gateway-api-install-crds >}}

## コントロールプレーンのインストール {#install-the-control-plane}

1 つまたは複数の `--set <parameter>=<value>` パラメータでデフォルト値を変更できます。
または、`--values <file>` パラメータでカスタム値ファイルを指定して複数のパラメータを設定できます。

{{< tip >}}
`helm show values <chart>` コマンドで各チャートのデフォルト値を表示できます。
または、Artifact Hub のチャートドキュメントで [base](https://artifacthub.io/packages/helm/istio-official/base?modal=values)、
[istiod](https://artifacthub.io/packages/helm/istio-official/istiod?modal=values)、
[CNI](https://artifacthub.io/packages/helm/istio-official/cni?modal=values)、
[ztunnel](https://artifacthub.io/packages/helm/istio-official/ztunnel?modal=values)、
[Gateway](https://artifacthub.io/packages/helm/istio-official/gateway?modal=values) の各チャートの設定パラメータも参照できます。
{{< /tip >}}

Helm インストールの使い方やカスタマイズの詳細は、[サイドカーインストールドキュメント](/ja/docs/setup/install/helm/)を参照してください。

[istioctl](/ja/docs/ambient/install/istioctl/) のプロファイル（インストールまたは削除するコンポーネントをグループ化）とは異なり、
Helm のプロファイルは設定値のグループ化のみを行います。

### 基本コンポーネント {#base-components}

`base` チャートには、Istio のセットアップに必要な基本的な CRD とクラスターロールが含まれています。
このチャートは他の Istio コンポーネントより先にインストールする必要があります。

{{< text syntax=bash snip_id=install_base >}}
$ helm install istio-base istio/base -n istio-system --create-namespace --wait
{{< /text >}}

### istiod コントロールプレーン {#istiod-control-plane}

`istiod` チャートは、リビジョン付きの Istiod をインストールします。
Istiod は、メッシュ内のトラフィックをルーティングするためにプロキシを管理・構成するコントロールプレーンコンポーネントです。

{{< text syntax=bash snip_id=install_istiod >}}
$ helm install istiod istio/istiod --namespace istio-system --set profile=ambient --wait
{{< /text >}}

### CNI ノードエージェント {#cni-node-agent}

`cni` チャートは Istio CNI ノードエージェントをインストールします。このエージェントは Ambient メッシュに属する Pod を検出し、
Pod と ztunnel ノードエージェント（後でインストール）間のトラフィックリダイレクトを構成します。

{{< text syntax=bash snip_id=install_cni >}}
$ helm install istio-cni istio/cni -n istio-system --set profile=ambient --wait
{{< /text >}}

## データプレーンのインストール {#install-the-data-plane}

### ztunnel DaemonSet {#ztunnel-daemonset}

`ztunnel` チャートは、ztunnel DaemonSet（Istio Ambient モードのノードエージェントコンポーネント）をインストールします。

{{< text syntax=bash snip_id=install_ztunnel >}}
$ helm install ztunnel istio/ztunnel -n istio-system --wait
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

Kubernetes クラスターが正しい外部 IP を割り当てた `LoadBalancer` サービスタイプ（`type: LoadBalancer`）をサポートしていない場合、
無限に待機しないよう `--wait` オプションを外して上記コマンドを実行してください。ゲートウェイのインストールに関する詳細は[ゲートウェイのインストール](/ja/docs/setup/additional-setup/gateway/)を参照してください。

## 設定 {#configuration}

サポートされている設定オプションやドキュメントを確認するには、次のコマンドを実行してください：

{{< text syntax=bash >}}
$ helm show values istio/istiod
{{< /text >}}

## インストールの検証 {#verifying-the-installation}

### ワークロードの状態を検証する {#verifying-the-workload-status}

すべてのコンポーネントをインストールした後、次のコマンドで Helm デプロイの状態を確認できます：

{{< text syntax=bash snip_id=show_components >}}
$ helm ls -n istio-system
NAME NAMESPACE REVISION UPDATED STATUS CHART APP VERSION
istio-base istio-system 1 2024-04-17 22:14:45.964722028 +0000 UTC deployed base-{{< istio_full_version >}} {{< istio_full_version >}}
istio-cni istio-system 1 2024-04-17 22:14:45.964722028 +0000 UTC deployed cni-{{< istio_full_version >}} {{< istio_full_version >}}
istiod istio-system 1 2024-04-17 22:14:45.964722028 +0000 UTC deployed istiod-{{< istio_full_version >}} {{< istio_full_version >}}
ztunnel istio-system 1 2024-04-17 22:14:45.964722028 +0000 UTC deployed ztunnel-{{< istio_full_version >}} {{< istio_full_version >}}
{{< /text >}}

次のコマンドでデプロイ済み Pod の状態を確認できます：

{{< text syntax=bash snip_id=check_pods >}}
$ kubectl get pods -n istio-system
NAME READY STATUS RESTARTS AGE
istio-cni-node-g97z5 1/1 Running 0 10m
istiod-5f4c75464f-gskxf 1/1 Running 0 10m
ztunnel-c2z4s 1/1 Running 0 10m
{{< /text >}}

### サンプルアプリケーションでの検証 {#verifying-with-the-sample-application}

Helm で Ambient モードをインストールした後、[サンプルアプリケーションのデプロイ](/ja/docs/ambient/getting-started/deploy-sample-app/)ガイドに従ってサンプルアプリとイングレスゲートウェイをデプロイできます。
その後、[アプリケーションを Ambient メッシュに追加](/ja/docs/ambient/getting-started/secure-and-visualize/#add-bookinfo-to-the-mesh)できます。

## アンインストール {#uninstall}

上記でインストールしたチャートをアンインストールすることで、Istio およびそのコンポーネントを削除できます。

1. `istio-system` 名前空間にインストールされているすべての Istio チャートを一覧表示：

   {{< text syntax=bash >}}
   $ helm ls -n istio-system
   NAME NAMESPACE REVISION UPDATED STATUS CHART APP VERSION
   istio-base istio-system 1 2024-04-17 22:14:45.964722028 +0000 UTC deployed base-{{< istio_full_version >}} {{< istio_full_version >}}
   istio-cni istio-system 1 2024-04-17 22:14:45.964722028 +0000 UTC deployed cni-{{< istio_full_version >}} {{< istio_full_version >}}
   istiod istio-system 1 2024-04-17 22:14:45.964722028 +0000 UTC deployed istiod-{{< istio_full_version >}} {{< istio_full_version >}}
   ztunnel istio-system 1 2024-04-17 22:14:45.964722028 +0000 UTC deployed ztunnel-{{< istio_full_version >}} {{< istio_full_version >}}
   {{< /text >}}

1.（オプション）すべての Istio ゲートウェイチャートのインストールを削除：

    {{< text syntax=bash snip_id=delete_ingress >}}
    $ helm delete istio-ingress -n istio-ingress
    $ kubectl delete namespace istio-ingress
    {{< /text >}}

1. ztunnel チャートを削除：

   {{< text syntax=bash snip_id=delete_ztunnel >}}
   $ helm delete ztunnel -n istio-system
   {{< /text >}}

1. Istio CNI チャートを削除：

   {{< text syntax=bash snip_id=delete_cni >}}
   $ helm delete istio-cni -n istio-system
   {{< /text >}}

1. istiod コントロールプレーンチャートを削除：

   {{< text syntax=bash snip_id=delete_istiod >}}
   $ helm delete istiod -n istio-system
   {{< /text >}}

1. Istio base チャートを削除：

   {{< tip >}}
   設計上、Helm でチャートを削除しても、そのチャートでインストールされたカスタムリソース定義（CRD）は削除されません。
   {{< /tip >}}

   {{< text syntax=bash snip_id=delete_base >}}
   $ helm delete istio-base -n istio-system
   {{< /text >}}

1. Istio でインストールされた CRD を削除（オプション）

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

## インストール前のマニフェスト生成 {#generate-a-manifest-before-installation}

Istio をインストールする前に、`helm template` サブコマンドを使って各コンポーネントのマニフェストを生成できます。
たとえば、`istiod` コンポーネント用のマニフェストを生成し、`kubectl` でインストールするには：

{{< text syntax=bash snip_id=none >}}
$ helm template istiod istio/istiod -n istio-system --kube-version {Kubernetes version of target cluster} > istiod.yaml
{{< /text >}}

生成されたマニフェストは、何がインストールされるかの確認や、マニフェストの変更履歴の追跡に利用できます。

{{< tip >}}
インストール時に使う他のフラグやカスタム値の上書きも、`helm template` コマンドに渡す必要があります。
{{< /tip >}}

上記で生成したマニフェストを使って、ターゲットクラスタに `istiod` コンポーネントを作成できます：

{{< text syntax=bash snip_id=none >}}
$ kubectl apply -f istiod.yaml
{{< /text >}}

{{< warning >}}
`helm template` で Istio をインストール・管理する場合、以下の注意点があります：

1. Istio の名前空間（デフォルトは `istio-system`）は手動で作成する必要があります。

1. リソースは `helm install` と同じ依存順でインストールされない場合があります。

1. この方法は Istio のリリースでテストされていません。

1. `helm install` は Kubernetes コンテキストから環境固有の設定を自動検出しますが、
   `helm template` はオフラインで動作するためそれができず、予期しない結果になる場合があります。特に、Kubernetes 環境がサードパーティサービスアカウントトークンをサポートしていない場合、[これらの手順](/ja/docs/ops/best-practices/security/#configure-third-party-service-account-tokens)に従う必要があります。

1. クラスタ内のリソースが正しい順序で利用できない場合、生成されたマニフェストの `kubectl apply` で一時的なエラーが発生することがあります。

1. `helm install` は設定変更時に削除すべきリソース（例：ゲートウェイの削除）を自動で削除しますが、
   `helm template` と `kubectl` の組み合わせでは自動削除されないため、手動で削除する必要があります。

{{< /warning >}}
