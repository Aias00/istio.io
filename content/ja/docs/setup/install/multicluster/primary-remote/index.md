---
title: プライマリ・リモート構成のインストール
description: プライマリ・リモートクラスタにまたがる Istio メッシュのインストール。
weight: 20
icon: setup
keywords: [kubernetes, multicluster]
test: no
owner: istio/wg-environments-maintainers
---

このガイドに従って、`cluster1` {{< gloss "primary cluster" >}}プライマリクラスタ{{< /gloss >}} に Istio コントロールプレーンをインストールし、`cluster2` {{< gloss "remote cluster" >}}リモートクラスタ{{< /gloss >}} を `cluster1` のコントロールプレーンに接続します。両方のクラスタは `network1` 上で動作しているため、両クラスタの Pod は直接通信できます。

インストールを続行する前に、[事前準備](/ja/docs/setup/install/multicluster/before-you-begin)の手順を完了していることを確認してください。

{{< boilerplate multi-cluster-with-metallb >}}

{{< warning >}}
これらの手順は AWS EKS プライマリクラスタのデプロイには対応していません。
この非互換性の理由は、AWS ロードバランサー（LB）が完全修飾ドメイン名（FQDN）として提供される一方、リモートクラスタは Kubernetes サービスタイプ 'ExternalName' を使用するためです。
しかし、'ExternalName' タイプは IP アドレスのみをサポートし、FQDN には対応していません。
{{< /warning >}}

この構成では、`cluster1` が両方のクラスタ API サーバーのサービスエンドポイントを監視します。
この方法により、コントロールプレーンは両方のクラスタのワークロードにサービスディスカバリを提供できます。

サービスのワークロード（Pod 間）はクラスタ間の境界を越えて直接通信します。

`cluster2` のサービスは、専用の[イーストウエスト](https://en.wikipedia.org/wiki/East-west_traffic)ゲートウェイ経由で `cluster1` のコントロールプレーンにトラフィックを送ります。

{{< image width="75%"
    link="arch.svg"
    caption="同一ネットワーク上のプライマリ・リモートクラスタ"
    >}}

## `cluster1` をプライマリクラスタとして構成する {#configure-cluster1-as-a-primary}

`cluster1` 用の `istioctl` 設定を作成します：

{{< tabset category-name="multicluster-primary-remote-install-type-primary-cluster" >}}

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
externalIstiod: true
EOF
{{< /text >}}

設定ファイルを `cluster1` に適用します：

{{< text bash >}}
$ istioctl install --context="${CTX_CLUSTER1}" -f cluster1.yaml
{{< /text >}}

`values.global.externalIstiod` を `true` に設定すると、`cluster1` 上のコントロールプレーンは他のリモートクラスタの外部コントロールプレーンとしても機能します。
この機能が有効な場合、`istiod` はリーダーシップロックを取得し、[適切なアノテーション](#set-the-control-plane-cluster-for-cluster2)が付与されたリモートクラスタ（この例では `cluster2`）を管理します。

{{< /tab >}}

{{< tab name="Helm" category-value="helm" >}}

以下の Helm コマンドを使用して、`cluster1` に Istio をプライマリとしてインストールします：

`cluster1` に `base` Chart をインストールします：

{{< text bash >}}
$ helm install istio-base istio/base -n istio-system --kube-context "${CTX_CLUSTER1}"
{{< /text >}}

次に、以下のマルチクラスタ設定を使用して `cluster1` に `istiod` Chart をインストールします：

{{< text bash >}}
$ helm install istiod istio/istiod -n istio-system --kube-context "${CTX_CLUSTER1}" --set global.meshID=mesh1 --set global.externalIstiod=true --set global.multiCluster.clusterName=cluster1 --set global.network=network1
{{< /text >}}

`values.global.externalIstiod` を `true` に設定してください。
これにより、`cluster1` 上のコントロールプレーンは他のリモートクラスタの外部コントロールプレーンとしても機能します。
この機能が有効な場合、`istiod` はリーダーロックを取得し、[適切なアノテーション](#set-the-control-plane-cluster-for-cluster2)が付与されたリモートクラスタ（この例では `cluster2`）を管理します。

{{< /tab >}}

{{< /tabset >}}

## `cluster1` にイーストウエストゲートウェイをインストールする {#install-the-east-west-gateway-in-cluster1}

`cluster1` にイーストウエストトラフィック専用のゲートウェイをインストールします。デフォルトでは、このゲートウェイはインターネットに公開されます。
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

## `cluster1` のコントロールプレーンを公開する {#expose-the-control-plane-in-cluster1}

`cluster2` をインストールする前に、`cluster1` のコントロールプレーンを公開し、`cluster2` のサービスがサービスディスカバリにアクセスできるようにします：

{{< text bash >}}
$ kubectl apply --context="${CTX_CLUSTER1}" -n istio-system -f \
 @samples/multicluster/expose-istiod.yaml@
{{< /text >}}

{{< warning >}}
コントロールプレーンにバージョン `rev` を指定している場合は、次のコマンドを実行してください：

{{< text bash >}}
$ sed 's/{{.Revision}}/rev/g' @samples/multicluster/expose-istiod-rev.yaml.tmpl@ | kubectl apply --context="${CTX_CLUSTER1}" -n istio-system -f -
{{< /text >}}

{{< /warning >}}

## `cluster2` のコントロールプレーンクラスタを設定する {#set-the-control-plane-cluster-for-cluster2}

`istio-system` 名前空間にアノテーションを追加して、`cluster2` を管理する外部コントロールプレーンを識別します：

{{< text bash >}}
$ kubectl --context="${CTX_CLUSTER2}" create namespace istio-system
$ kubectl --context="${CTX_CLUSTER2}" annotate namespace istio-system topology.istio.io/controlPlaneClusters=cluster1
{{< /text >}}

## `cluster2` をリモートクラスタとして構成する {#configure-cluster2-as-a-remote}

`cluster1` のイーストウエストゲートウェイのアドレスを保存します。

{{< text bash >}}
$ export DISCOVERY_ADDRESS=$(kubectl \
    --context="${CTX_CLUSTER1}" \
 -n istio-system get svc istio-eastwestgateway \
 -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
{{< /text >}}

次に、`cluster2` 用のリモートクラスタ設定を作成します：

{{< tabset category-name="multicluster-primary-remote-install-type-remote-cluster" >}}

{{< tab name="IstioOperator" category-value="iop" >}}

{{< text bash >}}
$ cat <<EOF > cluster2.yaml
apiVersion: install.istio.io/v1alpha1
kind: IstioOperator
spec:
profile: remote
values:
global:
meshID: mesh1
multiCluster:
clusterName: cluster2
network: network1
remotePilotAddress: ${DISCOVERY_ADDRESS}
EOF
{{< /text >}}

設定を `cluster2` に適用します：

{{< text bash >}}
$ istioctl install --context="${CTX_CLUSTER2}" -f cluster2.yaml
{{< /text >}}

{{< /tab >}}
{{< tab name="Helm" category-value="helm" >}}

以下の Helm コマンドを使用して、`cluster2` に Istio をリモートとしてインストールします：

`cluster2` に `base` Chart をインストールします：

{{< text bash >}}
$ helm install istio-base istio/base -n istio-system --set profile=remote --kube-context "${CTX_CLUSTER2}"
{{< /text >}}

次に、以下のマルチクラスタ設定を使用して `cluster2` に `istiod` Chart をインストールします：

{{< text bash >}}
$ helm install istiod istio/istiod -n istio-system --set profile=remote --set global.multiCluster.clusterName=cluster2 --set istiodRemote.injectionPath=/inject/cluster/cluster2/net/network1 --set global.configCluster=true --set global.remotePilotAddress="${DISCOVERY_ADDRESS}" --kube-context "${CTX_CLUSTER2}"
{{< /text >}}

{{< tip >}}
Istio バージョン 1.24 以降でのみ、`base` および `istiod` Helm Chart の `remote` プロファイルが利用可能です。
{{< /tip >}}

{{< /tab >}}

{{< /tabset >}}

{{< tip >}}
デモのため、ここでは `injectionPath` と `remotePilotAddress` パラメータでコントロールプレーンの場所を指定していますが、本番環境では正しく署名された DNS 証明書を使って `injectionURL` パラメータを設定することを推奨します。
詳細は[外部コントロールプレーンの説明](/ja/docs/setup/install/external-controlplane/#register-the-new-cluster)を参照してください。
{{< /tip >}}

## `cluster2` を `cluster1` のリモートクラスタとして接続する {#attach-cluster2-as-a-remote-cluster-of-cluster1}

リモートクラスタをコントロールプレーンに接続するため、`cluster1` のコントロールプレーンが `cluster2` の API サーバーにアクセスできるようにします。
これにより、以下が可能になります：

- コントロールプレーンが `cluster2` で動作するワークロードからの接続リクエストを検証できるようになります。
  API サーバーへのアクセス権がない場合、コントロールプレーンはリクエストを拒否します。

- `cluster2` で動作するサービスエンドポイントのディスカバリが有効になります。

また、`topology.istio.io/controlPlaneClusters` 名前空間アノテーションに含まれているため、
`cluster1` 上のコントロールプレーンは次のことも行います：

- `cluster2` の Webhook 証明書を修正します。

- 名前空間コントローラーを起動し、`cluster2` の名前空間に ConfigMap を書き込みます。

API サーバーが `cluster2` にアクセスできるようにするため、リモート Secret を生成し、`cluster1` に適用します：

{{< text bash >}}
$ istioctl create-remote-secret \
 --context="${CTX_CLUSTER2}" \
    --name=cluster2 | \
    kubectl apply -f - --context="${CTX_CLUSTER1}"
{{< /text >}}

**おめでとうございます！** プライマリ・リモートクラスタにまたがる Istio メッシュのインストールに成功しました！

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
