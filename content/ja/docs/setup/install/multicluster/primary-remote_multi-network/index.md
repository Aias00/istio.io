---
title: ネットワークをまたぐプライマリ・リモートクラスタのインストール
description: ネットワークをまたぎ、プライマリ・リモートクラスタに Istio メッシュをインストールします。
weight: 40
keywords: [kubernetes, multicluster]
test: no
owner: istio/wg-environments-maintainers
---

このガイドに従って、`cluster1` {{< gloss "primary cluster" >}}プライマリクラスタ{{< /gloss >}}に
Istio コントロールプレーンをインストールし、`cluster2`
{{< gloss "remote cluster" >}}リモートクラスタ{{< /gloss >}}を `cluster1` のコントロールプレーンに接続します。
クラスタ `cluster1` は `network1` 上、クラスタ `cluster2` は `network2` 上にあります。
そのため、クラスタ間の Pod 同士は直接通信できません。

インストールを続ける前に、[事前準備](/zh/docs/setup/install/multicluster/before-you-begin)の手順を完了していることを確認してください。

{{< boilerplate multi-cluster-with-metallb >}}

この構成では、クラスタ `cluster1` が両方のクラスタ API サーバのサービスエンドポイントを監視します。
この方法により、コントロールプレーンは両方のクラスタ内のワークロードにサービスディスカバリを提供できます。

クラスタ間のサービス通信は、専用のイーストウエストゲートウェイを介して間接的に行われます。
各クラスタのゲートウェイは、他のクラスタからアクセス可能でなければなりません。

`cluster2` のサービスは、同じイーストウエストゲートウェイを通じて `cluster1` のコントロールプレーンにアクセスします。

{{< image width="75%"
    link="arch.svg"
    caption="ネットワークをまたぐプライマリ・リモートクラスタ"
    >}}

## `cluster1` のデフォルトネットワークを設定する {#set-the-default-network-for-cluster1}

istio-system ネームスペースを作成した後、クラスタのネットワークを設定します：

{{< text bash >}}
$ kubectl --context="${CTX_CLUSTER1}" get namespace istio-system && \
  kubectl --context="${CTX_CLUSTER1}" label namespace istio-system topology.istio.io/network=network1
{{< /text >}}

## `cluster1` をプライマリクラスタとして設定する {#configure-cluster1-as-a-primary}

`cluster1` 用の `istioctl` 設定を作成します：

{{< tabset category-name="multicluster-primary-remote-install-type-primary-cluster" >}}

{{< tab name="IstioOperator" category-value="iop" >}}

istioctl と `IstioOperator` API を使って、`cluster1` に Istio をプライマリとしてインストールします。

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

設定を `cluster1` に適用します：

{{< text bash >}}
$ istioctl install --context="${CTX_CLUSTER1}" -f cluster1.yaml
{{< /text >}}

`values.global.externalIstiod` が `true` に設定されていることに注意してください。
これにより、`cluster1` 上にインストールされたコントロールプレーンが、他のリモートクラスタの外部コントロールプレーンとしても機能します。
この機能を有効にすると、`istiod` はリーダー選出ロックを取得しようとし、
[適切なアノテーションが付与された](#set-the-control-plane-cluster-for-cluster2)リモートクラスタ（この例では `cluster2`）を管理します。

{{< /tab >}}

{{< tab name="Helm" category-value="helm" >}}

以下の Helm コマンドを使って、`cluster1` に Istio をプライマリとしてインストールします：

`cluster1` に `base` チャートをインストールします：

{{< text bash >}}
$ helm install istio-base istio/base -n istio-system --kube-context "${CTX_CLUSTER1}"
{{< /text >}}

次に、以下のマルチクラスタ設定で `cluster1` に `istiod` チャートをインストールします：

{{< text bash >}}
$ helm install istiod istio/istiod -n istio-system --kube-context "${CTX_CLUSTER1}" --set global.meshID=mesh1 --set global.externalIstiod=true --set global.multiCluster.clusterName=cluster1 --set global.network=network1
{{< /text >}}

`values.global.externalIstiod` が `true` に設定されていることに注意してください。
これにより、`cluster1` 上にインストールされたコントロールプレーンが、他のリモートクラスタの外部コントロールプレーンとしても機能します。
この機能を有効にすると、`istiod` はリーダーロックを取得し、
[適切なアノテーションが付与された](#set-the-control-plane-cluster-for-cluster2)リモートクラスタ（この例では `cluster2`）を管理します。

{{< /tab >}}

{{< /tabset >}}

## `cluster1` にイーストウエストゲートウェイをインストールする {#install-the-east-west-gateway-in-cluster1}

`cluster1` に専用のイーストウエストトラフィックゲートウェイをインストールします。
デフォルトでは、このゲートウェイはインターネットに公開されます。
本番環境では、外部からの攻撃を防ぐために追加のアクセス制限（ファイアウォールルールなど）が必要な場合があります。
ご利用のクラウドプロバイダーに、利用可能なオプションについてご相談ください。

{{< tabset category-name="east-west-gateway-install-type-cluster-1" >}}

{{< tab name="IstioOperator" category-value="iop" >}}

{{< text bash >}}
$ @samples/multicluster/gen-eastwest-gateway.sh@ \
 --network network1 | \
 istioctl --context="${CTX_CLUSTER1}" install -y -f -
{{< /text >}}

{{< warning >}}
コントロールプレーンがリビジョン付きでインストールされている場合は、`gen-eastwest-gateway.sh` コマンドに
`--revision rev` フラグを追加してください。
{{< /warning >}}

{{< /tab >}}
{{< tab name="Helm" category-value="helm" >}}

以下の Helm コマンドを使って、`cluster1` にイーストウエストゲートウェイをインストールします：

{{< text bash >}}
$ helm install istio-eastwestgateway istio/gateway -n istio-system --kube-context "${CTX_CLUSTER1}" --set name=istio-eastwestgateway --set networkGateway=network1
{{< /text >}}

{{< warning >}}
コントロールプレーンがリビジョン付きでインストールされている場合は、Helm インストールコマンドに
`--set revision=<my-revision>` フラグを追加してください。
{{< /warning >}}

{{< /tab >}}

{{< /tabset >}}

イーストウエストゲートウェイが外部 IP アドレスを取得するまで待ちます：

{{< text bash >}}
$ kubectl --context="${CTX_CLUSTER1}" get svc istio-eastwestgateway -n istio-system
NAME TYPE CLUSTER-IP EXTERNAL-IP PORT(S) AGE
istio-eastwestgateway LoadBalancer 10.80.6.124 34.75.71.237 ... 51s
{{< /text >}}

## `cluster1` のコントロールプレーンを公開する {#expose-the-control-plane-in-cluster1}

`cluster2` をインストールする前に、`cluster1` のコントロールプレーンを公開し、
`cluster2` のサービスがサービスディスカバリにアクセスできるようにします。

{{< text bash >}}
$ kubectl apply --context="${CTX_CLUSTER1}" -n istio-system -f \
 @samples/multicluster/expose-istiod.yaml@
{{< /text >}}

{{< warning >}}
コントロールプレーンにリビジョン `rev` が指定されている場合は、次のコマンドを実行してください：

{{< text bash >}}
$ sed 's/{{.Revision}}/rev/g' @samples/multicluster/expose-istiod-rev.yaml.tmpl@ | kubectl apply --context="${CTX_CLUSTER1}" -n istio-system -f -
{{< /text >}}

{{< /warning >}}

## `cluster2` のコントロールプレーンクラスタを設定する {#set-the-control-plane-cluster-for-cluster2}

istio-system ネームスペースを作成した後、クラスタのネットワークを設定します：
`istio-system` ネームスペースにアノテーションを追加して、どの外部コントロールプレーンクラスタが `cluster2` を管理するかを識別します：

{{< text bash >}}
$ kubectl --context="${CTX_CLUSTER2}" create namespace istio-system
$ kubectl --context="${CTX_CLUSTER2}" annotate namespace istio-system topology.istio.io/controlPlaneClusters=cluster1
{{< /text >}}

`topology.istio.io/controlPlaneClusters` ネームスペースアノテーションを
`cluster1` に設定することで、`cluster1` 上の同じネームスペース（この例では istio-system）にある
`istiod` が [リモートクラスタとして接続された](#attach-cluster2-as-a-remote-cluster-of-cluster1) `cluster2` を管理するよう指示します。

## `cluster2` のデフォルトネットワークを設定する {#set-the-default-network-for-cluster2}

`istio-system` ネームスペースにラベルを追加して、`cluster2` のネットワークを設定します：

{{< text bash >}}
$ kubectl --context="${CTX_CLUSTER2}" label namespace istio-system topology.istio.io/network=network2
{{< /text >}}

## `cluster2` をリモートクラスタとして設定する {#configure-cluster2-as-a-remote}

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
istiodRemote:
injectionPath: /inject/cluster/cluster2/net/network2
global:
remotePilotAddress: ${DISCOVERY_ADDRESS}
EOF
{{< /text >}}

設定を `cluster2` に適用します：

{{< text bash >}}
$ istioctl install --context="${CTX_CLUSTER2}" -f cluster2.yaml
{{< /text >}}

{{< /tab >}}
{{< tab name="Helm" category-value="helm" >}}

以下の Helm コマンドを使って、`cluster2` に Istio をリモートとしてインストールします：

`cluster2` に `base` チャートをインストールします：

{{< text bash >}}
$ helm install istio-base istio/base -n istio-system --set profile=remote --kube-context "${CTX_CLUSTER2}"
{{< /text >}}

次に、以下のマルチクラスタ設定で `cluster2` に `istiod` チャートをインストールします：

{{< text bash >}}
$ helm install istiod istio/istiod -n istio-system --set profile=remote --set global.multiCluster.clusterName=cluster2 --set global.network=network2 --set istiodRemote.injectionPath=/inject/cluster/cluster2/net/network2 --set global.configCluster=true --set global.remotePilotAddress="${DISCOVERY_ADDRESS}" --kube-context "${CTX_CLUSTER2}"
{{< /text >}}

{{< tip >}}
Istio バージョン 1.24 以降のみ、`base` および `istiod` Helm チャートの `remote` プロファイルが利用可能です。
{{< /tip >}}

{{< /tab >}}

{{< /tabset >}}

{{< tip >}}
ここでは `injectionPath` と `remotePilotAddress` パラメータでコントロールプレーンの場所を設定しています。
これはデモのための簡易設定ですが、本番環境では、[外部コントロールプレーンの説明](/zh/docs/setup/install/external-controlplane/#register-the-new-cluster)のように、正しく署名された DNS 証明書を使って `injectionURL` パラメータを設定することを推奨します。
{{< /tip >}}

## `cluster1` のリモートクラスタとして `cluster2` を接続する {#attach-cluster2-as-a-remote-cluster-of-cluster1}

リモートクラスタをコントロールプレーンに接続するために、`cluster1` のコントロールプレーンが
`cluster2` の API サーバにアクセスできるようにします。これにより、以下が可能になります：

- コントロールプレーンが `cluster2` で動作するワークロードからの接続リクエストを検証できるようになります。
  API サーバへのアクセス権がない場合、このコントロールプレーンはこれらのリクエストを拒否します。

- `cluster2` で動作するサービスエンドポイントのディスカバリが有効になります。

`topology.istio.io/controlPlaneClusters` ネームスペースアノテーションに含まれているため、
`cluster1` 上のコントロールプレーンはさらに：

- `cluster2` の Webhook の証明書を修正します。

- ネームスペースコントローラーを起動し、`cluster2` のネームスペースに ConfigMap を書き込みます。

API サーバへのアクセスを `cluster2` に許可するために、
リモートシークレットを生成し、それを `cluster1` に適用します：

{{< text bash >}}
$ istioctl create-remote-secret \
 --context="${CTX_CLUSTER2}" \
    --name=cluster2 | \
    kubectl apply -f - --context="${CTX_CLUSTER1}"
{{< /text >}}

## `cluster2` にイーストウエストゲートウェイをインストールする {#install-the-east-west-gateway-in-cluster2}

上記の `cluster1` と同様に、`cluster2` にイーストウエストトラフィック専用のゲートウェイをインストールし、ユーザーサービスを公開します。

{{< tabset category-name="east-west-gateway-install-type-cluster-2" >}}

{{< tab name="IstioOperator" category-value="iop" >}}

{{< text bash >}}
$ @samples/multicluster/gen-eastwest-gateway.sh@ \
 --network network2 | \
 istioctl --context="${CTX_CLUSTER2}" install -y -f -
{{< /text >}}

{{< /tab >}}
{{< tab name="Helm" category-value="helm" >}}

以下の Helm コマンドを使って、`cluster2` にイーストウエストゲートウェイをインストールします：

{{< text bash >}}
$ helm install istio-eastwestgateway istio/gateway -n istio-system --kube-context "${CTX_CLUSTER2}" --set name=istio-eastwestgateway --set networkGateway=network2
{{< /text >}}

{{< warning >}}
コントロールプレーンがリビジョン付きでインストールされている場合は、Helm インストールコマンドに
`--set revision=<my-revision>` を追加してください。
{{< /warning >}}

{{< /tab >}}

{{< /tabset >}}

イーストウエストゲートウェイが外部 IP アドレスを取得するまで待ちます：

{{< text bash >}}
$ kubectl --context="${CTX_CLUSTER2}" get svc istio-eastwestgateway -n istio-system
NAME TYPE CLUSTER-IP EXTERNAL-IP PORT(S) AGE
istio-eastwestgateway LoadBalancer 10.0.12.121 34.122.91.98 ... 51s
{{< /text >}}

## `cluster1` と `cluster2` のサービスを公開する {#expose-services-in-cluster1-and-cluster2}

クラスタが異なるネットワークにあるため、両方のクラスタのイーストウエストゲートウェイ上で全てのユーザーサービス（\*.local）を公開する必要があります。
このゲートウェイはインターネットに公開されていますが、その背後のサービスには、信頼できる mTLS 証明書とワークロード ID を持つサービスのみがアクセスできます。
これは、同じネットワーク内にあるかのように動作します。

{{< text bash >}}
$ kubectl --context="${CTX_CLUSTER1}" apply -n istio-system -f \
 @samples/multicluster/expose-services.yaml@
{{< /text >}}

{{< tip >}}
`cluster2` がリモートプロファイルでインストールされているため、プライマリクラスタでサービスを公開すると、両方のクラスタのイーストウエストゲートウェイでこれらのサービスが公開されます。
{{< /tip >}}

**おめでとうございます！** ネットワークをまたぐプライマリ・リモートクラスタに Istio メッシュをインストールできました。

## 次のステップ {#next-steps}

[インストール結果を検証する](/zh/docs/setup/install/multicluster/verify)ことができます。

## クリーンアップ {#cleanup}

Istio をインストールしたのと同じ方法（istioctl または Helm）で、
`cluster1` と `cluster2` から Istio をアンインストールします。

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

`cluster1` から `istio-system` ネームスペースを削除します：

{{< text syntax=bash >}}
$ kubectl delete ns istio-system --context="${CTX_CLUSTER1}"
{{< /text >}}

`cluster2` から Istio Helm インストールを削除します：

{{< text syntax=bash >}}
$ helm delete istiod -n istio-system --kube-context "${CTX_CLUSTER2}"
$ helm delete istio-eastwestgateway -n istio-system --kube-context "${CTX_CLUSTER2}"
$ helm delete istio-base -n istio-system --kube-context "${CTX_CLUSTER2}"
{{< /text >}}

`cluster2` から `istio-system` ネームスペースを削除します：

{{< text syntax=bash >}}
$ kubectl delete ns istio-system --context="${CTX_CLUSTER2}"
{{< /text >}}

（オプション）Istio インストールの CRD を削除します：

CRD を削除すると、クラスタ内で作成したすべての Istio リソースが永久に削除されます。
クラスタにインストールされている Istio CRD を削除するには、次のコマンドを実行します：

{{< text syntax=bash snip_id=delete_crds >}}
$ kubectl get crd -oname --context "${CTX_CLUSTER1}" | grep --color=never 'istio.io' | xargs kubectl delete --context "${CTX_CLUSTER1}"
$ kubectl get crd -oname --context "${CTX_CLUSTER2}" | grep --color=never 'istio.io' | xargs kubectl delete --context "${CTX_CLUSTER2}"
{{< /text >}}

{{< /tab >}}

{{< /tabset >}}
