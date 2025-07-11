---
title: 異なるネットワーク上でのマルチプライマリ構成のインストール
description: 異なるネットワーク・マルチプライマリ構成の Istio メッシュインストール。
weight: 30
icon: setup
keywords: [kubernetes, multicluster]
test: yes
owner: istio/wg-environments-maintainers
---

このガイドに従って、`cluster1` と `cluster2` の両方のクラスタに Istio コントロールプレーンをインストールし、両方をプライマリクラスタ（{{< gloss >}}プライマリクラスタ{{< /gloss >}}）として設定します。
クラスタ `cluster1` は `network1`、クラスタ `cluster2` は `network2` 上にあります。
これは、クラスタ間の Pod 同士が直接通信できないことを意味します。

インストールを続行する前に、[事前準備](/ja/docs/setup/install/multicluster/before-you-begin)の手順を完了していることを確認してください。

{{< boilerplate multi-cluster-with-metallb >}}

この構成では、`cluster1` と `cluster2` の両方が両クラスタの API サーバーのサービスエンドポイントを監視します。

クラスタ間のサービスワークロードは、専用の[イーストウエスト](https://en.wikipedia.org/wiki/East-west_traffic)ゲートウェイを介して間接的に通信します。各クラスタのゲートウェイは、他のクラスタからアクセス可能でなければなりません。

{{< image width="75%"
    link="arch.svg"
    caption="異なるネットワーク上のマルチプライマリクラスタ"
    >}}

## `cluster1` のデフォルトネットワークを設定する {#set-the-default-network-for-cluster1}

`istio-system` 名前空間を作成した後、クラスタのネットワークを設定します：

{{< text bash >}}
$ kubectl --context="${CTX_CLUSTER1}" get namespace istio-system && \
  kubectl --context="${CTX_CLUSTER1}" label namespace istio-system topology.istio.io/network=network1
{{< /text >}}

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

## `cluster1` にイーストウエストゲートウェイをインストールする {#install-the-east-west-gateway-in-cluster1}

`cluster1` に専用の[イーストウエスト](https://en.wikipedia.org/wiki/East-west_traffic)ゲートウェイをインストールします。
デフォルトでは、このゲートウェイはインターネットに公開されます。
本番環境では、外部からの攻撃を防ぐために追加のアクセス制限（例：ファイアウォールルール）が必要な場合があります。
ご利用のクラウドプロバイダーに、利用可能なオプションについてご相談ください。

{{< tabset category-name="east-west-gateway-install-type-cluster-1" >}}

{{< tab name="IstioOperator" category-value="iop" >}}

{{< text bash >}}
$ @samples/multicluster/gen-eastwest-gateway.sh@ \
 --network network1 | \
 istioctl --context="${CTX_CLUSTER1}" install -y -f -
{{< /text >}}

{{< warning >}}
コントロールプレーンがすでにリビジョン付きでインストールされている場合は、`gen-eastwest-gateway.sh` コマンドに `--revision rev` フラグを追加できます。
{{< /warning >}}

{{< /tab >}}
{{< tab name="Helm" category-value="helm" >}}

以下の Helm コマンドを使用して、`cluster1` にイーストウエストゲートウェイをインストールします：

{{< text bash >}}
$ helm install istio-eastwestgateway istio/gateway -n istio-system --kube-context "${CTX_CLUSTER1}" --set name=istio-eastwestgateway --set networkGateway=network1
{{< /text >}}

{{< warning >}}
コントロールプレーンがリビジョン付きでインストールされている場合は、Helm インストールコマンドに `--set revision=<my-revision>` フラグを追加する必要があります。
{{< /warning >}}

{{< /tab >}}

{{< /tabset >}}

イーストウエストゲートウェイに外部 IP アドレスが割り当てられるまで待ちます：

{{< text bash >}}
$ kubectl --context="${CTX_CLUSTER1}" get svc istio-eastwestgateway -n istio-system
NAME TYPE CLUSTER-IP EXTERNAL-IP PORT(S) AGE
istio-eastwestgateway LoadBalancer 10.80.6.124 34.75.71.237 ... 51s
{{< /text >}}

## `cluster1` のサービスを公開する {#expose-services-in-cluster1}

クラスタが異なるネットワーク上にあるため、両方のクラスタのイーストウエストゲートウェイで全サービス（\*.local）を公開する必要があります。
このゲートウェイはインターネットに公開されていますが、その背後のサービスには信頼された mTLS 証明書とワークロード ID を持つサービスのみがアクセスできます。
これは、同じネットワーク内にあるかのように動作します。

{{< text bash >}}
$ kubectl --context="${CTX_CLUSTER1}" apply -n istio-system -f \
 @samples/multicluster/expose-services.yaml@
{{< /text >}}

## `cluster2` のデフォルトネットワークを設定する {#set-the-default-network-for-cluster2}

istio-system 名前空間を作成した後、クラスタのネットワークを設定します：

{{< text bash >}}
$ kubectl --context="${CTX_CLUSTER2}" get namespace istio-system && \
  kubectl --context="${CTX_CLUSTER2}" label namespace istio-system topology.istio.io/network=network2
{{< /text >}}

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
network: network2
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
$ helm install istiod istio/istiod -n istio-system --kube-context "${CTX_CLUSTER2}" --set global.meshID=mesh1 --set global.multiCluster.clusterName=cluster2 --set global.network=network2
{{< /text >}}

{{< /tab >}}

{{< /tabset >}}

## `cluster2` にイーストウエストゲートウェイをインストールする {#install-the-east-west-gateway-in-cluster2}

上記の `cluster1` の手順と同様に、`cluster2` にイーストウエストトラフィック用のゲートウェイをインストールします。

{{< tabset category-name="east-west-gateway-install-type-cluster-2" >}}

{{< tab name="IstioOperator" category-value="iop" >}}

{{< text bash >}}
$ @samples/multicluster/gen-eastwest-gateway.sh@ \
 --network network2 | \
 istioctl --context="${CTX_CLUSTER2}" install -y -f -
{{< /text >}}

{{< /tab >}}
{{< tab name="Helm" category-value="helm" >}}

以下の Helm コマンドを使用して、`cluster2` にイーストウエストゲートウェイをインストールします：

{{< text bash >}}
$ helm install istio-eastwestgateway istio/gateway -n istio-system --kube-context "${CTX_CLUSTER2}" --set name=istio-eastwestgateway --set networkGateway=network2
{{< /text >}}

{{< warning >}}
コントロールプレーンがリビジョン付きでインストールされている場合は、Helm インストールコマンドに `--set revision=<my-revision>` フラグを追加する必要があります。
{{< /warning >}}

{{< /tab >}}

{{< /tabset >}}

イーストウエストゲートウェイに外部 IP アドレスが割り当てられるまで待ちます：

{{< text bash >}}
$ kubectl --context="${CTX_CLUSTER2}" get svc istio-eastwestgateway -n istio-system
NAME TYPE CLUSTER-IP EXTERNAL-IP PORT(S) AGE
istio-eastwestgateway LoadBalancer 10.0.12.121 34.122.91.98 ... 51s
{{< /text >}}

## `cluster2` のサービスを公開する {#expose-services-in-cluster2}

上記の `cluster1` の手順と同様に、イーストウエストゲートウェイを介してサービスを公開します。

{{< text bash >}}
$ kubectl --context="${CTX_CLUSTER2}" apply -n istio-system -f \
 @samples/multicluster/expose-services.yaml@
{{< /text >}}

## エンドポイントディスカバリの有効化 {#enable-endpoint-discovery}

`cluster2` に、`cluster1` の API サーバーへのアクセス権を提供するリモート Secret をインストールします。

{{< text bash >}}
$ istioctl create-remote-secret \
 --context="${CTX_CLUSTER1}" \
  --name=cluster1 | \
  kubectl apply -f - --context="${CTX_CLUSTER2}"
{{< /text >}}

`cluster1` に、`cluster2` の API サーバーへのアクセス権を提供するリモート Secret をインストールします。

{{< text bash >}}
$ istioctl create-remote-secret \
 --context="${CTX_CLUSTER2}" \
  --name=cluster2 | \
  kubectl apply -f - --context="${CTX_CLUSTER1}"
{{< /text >}}

**おめでとうございます！** 異なるネットワーク上のマルチプライマリ構成クラスタに Istio メッシュをインストールできました。

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
$ helm delete istio-eastwestgateway -n istio-system --kube-context "${CTX_CLUSTER1}"
$ helm delete istio-base -n istio-system --kube-context "${CTX_CLUSTER1}"
{{< /text >}}

`cluster1` から `istio-system` 名前空間を削除します：

{{< text syntax=bash >}}
$ kubectl delete ns istio-system --context="${CTX_CLUSTER1}"
{{< /text >}}

`cluster2` から Istio Helm インストールを削除します：

{{< text syntax=bash >}}
$ helm delete istiod -n istio-system --kube-context "${CTX_CLUSTER2}"
$ helm delete istio-eastwestgateway -n istio-system --kube-context "${CTX_CLUSTER2}"
$ helm delete istio-base -n istio-system --kube-context "${CTX_CLUSTER2}"
{{< /text >}}

`cluster2` から `istio-system` 名前空間を削除します：

{{< text syntax=bash >}}
$ kubectl delete ns istio-system --context="${CTX_CLUSTER2}"
{{< /text >}}

（オプション）Istio がインストールした CRD を削除します：

CRD を削除すると、クラスタ内で作成したすべての Istio リソースが完全に削除されます。
クラスタにインストールされた Istio CRD を削除するには、以下のコマンドを実行します：

{{< text syntax=bash snip_id=delete_crds >}}
$ kubectl get crd -oname --context "${CTX_CLUSTER1}" | grep --color=never 'istio.io' | xargs kubectl delete --context "${CTX_CLUSTER1}"
$ kubectl get crd -oname --context "${CTX_CLUSTER2}" | grep --color=never 'istio.io' | xargs kubectl delete --context "${CTX_CLUSTER2}"
{{< /text >}}

{{< /tab >}}

{{< /tabset >}}
