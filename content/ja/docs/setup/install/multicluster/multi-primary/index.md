---
title: マルチプライマリ構成のインストール
description: 複数のプライマリクラスタにまたがって Istio メッシュをインストールします。
weight: 10
icon: setup
keywords: [kubernetes, multicluster]
test: yes
owner: istio/wg-environments-maintainers
---

このガイドに従って、`cluster1` と `cluster2` の両方のクラスタに Istio コントロールプレーンをインストールし、各クラスタをプライマリクラスタ（{{< gloss >}}プライマリクラスタ{{< /gloss >}}）として設定します。
両方のクラスタはネットワーク `network1` 上で動作しているため、両クラスタの Pod は直接通信できます。

インストールを続行する前に、[事前準備](/ja/docs/setup/install/multicluster/before-you-begin)の手順を完了していることを確認してください。

この構成では、各コントロールプレーンが両方のクラスタ API サーバーのサービスエンドポイントを監視します。

サービスのワークロード（pod 間）はクラスタ間の境界を越えて直接通信します。

{{< image width="75%"
    link="arch.svg"
    caption="同一ネットワーク上のマルチプライマリクラスタ"
    >}}

## `cluster1` をプライマリクラスタとして構成する {#configure-cluster1-as-a-primary}

`cluster1` 用の `istioctl` 設定を作成します：

{{< tabset category-name="multicluster-install-type-cluster-1" >}}

{{< tab name="IstioOperator" category-value="iop" >}}

istioctl と `IstioOperator` API を使用して、`cluster1` に Istio をプライマリとしてインストールします。

{{< text bash >}}
$ cat <<EOF > cluster1.yaml
apiVersion: install.istio.io/v1alpha1
kind: IstioOperator
spec:
values:
global:
meshID: mesh1
multiCluster:
clusterName: cluster1
network: network1
EOF
{{< /text >}}

設定ファイルを `cluster1` に適用します：

{{< text bash >}}
$ istioctl install --context="${CTX_CLUSTER1}" -f cluster1.yaml
{{< /text >}}

{{< /tab >}}
{{< tab name="Helm" category-value="helm" >}}

以下の Helm コマンドを使用して、`cluster1` に Istio をプライマリとしてインストールします：

`cluster1` に `base` Chart をインストールします：

{{< text bash >}}
$ helm install istio-base istio/base -n istio-system --kube-context "${CTX_CLUSTER1}"
{{< /text >}}

次に、以下のマルチクラスタ設定を使用して `cluster1` に `istiod` Chart をインストールします：

{{< text bash >}}
$ helm install istiod istio/istiod -n istio-system --kube-context "${CTX_CLUSTER1}" --set global.meshID=mesh1 --set global.multiCluster.clusterName=cluster1 --set global.network=network1
{{< /text >}}

{{< /tab >}}

{{< /tabset >}}

## `cluster2` をプライマリクラスタとして構成する {#configure-cluster2-as-a-primary}

`cluster2` 用の `istioctl` 設定を作成します：

{{< tabset category-name="multicluster-install-type-cluster-2" >}}

{{< tab name="IstioOperator" category-value="iop" >}}

istioctl と `IstioOperator` API を使用して、`cluster2` に Istio をプライマリとしてインストールします。

{{< text bash >}}
$ cat <<EOF > cluster2.yaml
apiVersion: install.istio.io/v1alpha1
kind: IstioOperator
spec:
values:
global:
meshID: mesh1
multiCluster:
clusterName: cluster2
network: network1
EOF
{{< /text >}}

設定ファイルを `cluster2` に適用します：

{{< text bash >}}
$ istioctl install --context="${CTX_CLUSTER2}" -f cluster2.yaml
{{< /text >}}

{{< /tab >}}
{{< tab name="Helm" category-value="helm" >}}

以下の Helm コマンドを使用して、`cluster2` に Istio をプライマリとしてインストールします：

`cluster2` に `base` Chart をインストールします：

{{< text bash >}}
$ helm install istio-base istio/base -n istio-system --kube-context "${CTX_CLUSTER2}"
{{< /text >}}

次に、以下のマルチクラスタ設定を使用して `cluster2` に `istiod` Chart をインストールします：

{{< text bash >}}
$ helm install istiod istio/istiod -n istio-system --kube-context "${CTX_CLUSTER2}" --set global.meshID=mesh1 --set global.multiCluster.clusterName=cluster2 --set global.network=network1
{{< /text >}}

{{< /tab >}}

{{< /tabset >}}

## エンドポイントディスカバリの有効化 {#enable-endpoint-discovery}

`cluster2` に、`cluster1` の API サーバーへのアクセス権を提供するリモートクラスタの secret をインストールします。

{{< text bash >}}
$ istioctl create-remote-secret \
 --context="${CTX_CLUSTER1}" \
    --name=cluster1 | \
    kubectl apply -f - --context="${CTX_CLUSTER2}"
{{< /text >}}

`cluster1` に、`cluster2` の API サーバーへのアクセス権を提供するリモートクラスタの secret をインストールします。

{{< text bash >}}
$ istioctl create-remote-secret \
 --context="${CTX_CLUSTER2}" \
    --name=cluster2 | \
    kubectl apply -f - --context="${CTX_CLUSTER1}"
{{< /text >}}

**おめでとうございます！** 複数のプライマリクラスタにまたがる Istio メッシュのインストールに成功しました！

## 次のステップ {#next-steps}

これで、[インストールの検証](/ja/docs/setup/install/multicluster/verify)を行うことができます。

## クリーンアップ {#cleanup}

Istio のインストールに使用したのと同じ方法（istioctl または Helm）で、
`cluster1` および `cluster2` から Istio をアンインストールします。

{{< tabset category-name="multicluster-uninstall-type-cluster-1" >}}

{{< tab name="IstioOperator" category-value="iop" >}}

`cluster1` から Istio をアンインストールします：

{{< text syntax=bash snip_id=none >}}
$ istioctl uninstall --context="${CTX_CLUSTER1}" -y --purge
$ kubectl delete ns istio-system --context="${CTX_CLUSTER1}"
{{< /text >}}

`cluster2` から Istio をアンインストールします：

{{< text syntax=bash snip_id=none >}}
$ istioctl uninstall --context="${CTX_CLUSTER2}" -y --purge
$ kubectl delete ns istio-system --context="${CTX_CLUSTER2}"
{{< /text >}}

{{< /tab >}}

{{< tab name="Helm" category-value="helm" >}}

`cluster1` から Istio Helm インストールを削除します：

{{< text syntax=bash >}}
$ helm delete istiod -n istio-system --kube-context "${CTX_CLUSTER1}"
$ helm delete istio-base -n istio-system --kube-context "${CTX_CLUSTER1}"
{{< /text >}}

`cluster1` から `istio-system` 名前空間を削除します：

{{< text syntax=bash >}}
$ kubectl delete ns istio-system --context="${CTX_CLUSTER1}"
{{< /text >}}

`cluster2` から Istio Helm インストールを削除します：

{{< text syntax=bash >}}
$ helm delete istiod -n istio-system --kube-context "${CTX_CLUSTER2}"
$ helm delete istio-base -n istio-system --kube-context "${CTX_CLUSTER2}"
{{< /text >}}

`cluster2` から `istio-system` 名前空間を削除します：

{{< text syntax=bash >}}
$ kubectl delete ns istio-system --context="${CTX_CLUSTER2}"
{{< /text >}}

（オプション）Istio がインストールした CRD を削除します：

CRD を削除すると、クラスタ内で作成したすべての Istio リソースが完全に削除されます。
以下のコマンドでクラスタにインストールされた Istio CRD を削除します：

{{< text syntax=bash snip_id=delete_crds >}}
$ kubectl get crd -oname --context "${CTX_CLUSTER1}" | grep --color=never 'istio.io' | xargs kubectl delete --context "${CTX_CLUSTER1}"
$ kubectl get crd -oname --context "${CTX_CLUSTER2}" | grep --color=never 'istio.io' | xargs kubectl delete --context "${CTX_CLUSTER2}"
{{< /text >}}

{{< /tab >}}

{{< /tabset >}}
