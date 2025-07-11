---
title: mTLS 有効化の検証
description: Ambient メッシュでワークロード間の mTLS が有効化されていることを検証する方法を解説します。
weight: 15
owner: istio/wg-networking-maintainers
test: no
---

アプリケーションを Ambient メッシュに追加したら、以下のいずれかの方法でワークロード間で mTLS が有効になっているか簡単に検証できます：

## ワークロードの ztunnel 設定で mTLS を検証 {#validate-mtls-using-workloads-ztunnel-configurations}

便利な `istioctl ztunnel-config workloads` コマンドを使うと、`PROTOCOL` 列の値で HBONE トラフィックの送受信設定がされているか確認できます。例：

{{< text syntax=bash >}}
$ istioctl ztunnel-config workloads
NAMESPACE POD NAME IP NODE WAYPOINT PROTOCOL
default details-v1-857849f66-ft8wx 10.42.0.5 k3d-k3s-default-agent-0 None HBONE
default kubernetes 172.20.0.3 None TCP
default productpage-v1-c5b7f7dbc-hlhpd 10.42.0.8 k3d-k3s-default-agent-0 None HBONE
default ratings-v1-68d5f5486b-b5sbj 10.42.0.6 k3d-k3s-default-agent-0 None HBONE
default reviews-v1-7dc5fc4b46-ndrq9 10.42.1.5 k3d-k3s-default-agent-1 None HBONE
default reviews-v2-6cf45d556b-4k4md 10.42.0.7 k3d-k3s-default-agent-0 None HBONE
default reviews-v3-86cb7d97f8-zxzl4 10.42.1.6 k3d-k3s-default-agent-1 None HBONE
{{< /text >}}

ワークロードで HBONE が設定されていても、明文トラフィックを拒否するとは限りません。明文トラフィックを拒否したい場合は、`PeerAuthentication` ポリシーで mTLS モードを `STRICT` に設定してください。

## メトリクスで mTLS を検証 {#validate-mtls-from-metrics}

[Prometheus をインストール](/ja/docs/ops/integrations/prometheus/#installation)している場合、ポートフォワードして以下のコマンドで Prometheus UI を開けます：

{{< text syntax=bash >}}
$ istioctl dashboard prometheus
{{< /text >}}

Prometheus で TCP メトリクス値を確認できます。Graph を選択し、`istio_tcp_connections_opened_total`、`istio_tcp_connections_closed_total`、`istio_tcp_received_bytes_total`、`istio_tcp_sent_bytes_total` などのメトリクスを入力し、Execute をクリックします。データには以下のようなエントリが含まれます：

{{< text syntax=plain >}}
istio_tcp_connections_opened_total{
app="ztunnel",
connection_security_policy="mutual_tls",
destination_principal="spiffe://cluster.local/ns/default/sa/bookinfo-details",
destination_service="details.default.svc.cluster.local",
reporter="source",
request_protocol="tcp",
response_flags="-",
source_app="curl",
source_principal="spiffe://cluster.local/ns/default/sa/curl",source_workload_namespace="default",
...}
{{< /text >}}

`connection_security_policy` の値が `mutual_tls` であること、および期待するソース・ターゲットのアイデンティティ情報を確認してください。

## ログで mTLS を検証 {#validate-mtls-from-logs}

ピアアイデンティティ情報と合わせて、ソースまたはターゲットの ztunnel ログを確認し、mTLS が有効かどうかを確認できます。以下は `curl` サービスから `details` サービスへのリクエスト時のソース ztunnel のログ例です：

{{< text syntax=plain >}}
2024-08-21T15:32:05.754291Z info access connection complete src.addr=10.42.0.9:33772 src.workload="curl-7656cf8794-6lsm4" src.namespace="default"
src.identity="spiffe://cluster.local/ns/default/sa/curl" dst.addr=10.42.0.5:15008 dst.hbone_addr=10.42.0.5:9080 dst.service="details.default.svc.cluster.local"
dst.workload="details-v1-857849f66-ft8wx" dst.namespace="default" dst.identity="spiffe://cluster.local/ns/default/sa/bookinfo-details"
direction="outbound" bytes_sent=84 bytes_recv=358 duration="15ms"
{{< /text >}}

`src.identity` と `dst.identity` の値が正しいか確認してください。これらはソースワークロードとターゲットワークロード間の mTLS 通信に使われるアイデンティティです。詳細は[ztunnel トラフィックのログによる検証](/ja/docs/ambient/usage/troubleshoot-ztunnel/#verifying-ztunnel-traffic-through-logs)を参照してください。

## Kiali ダッシュボードで検証 {#validate-with-kiali-dashboard}

Kiali と Prometheus をインストールしている場合、Kiali のダッシュボードでワークロード間通信を可視化できます。ピアアイデンティティ情報と合わせて、任意のワークロード間の接続にロックアイコンが表示されていれば mTLS が有効化されていることを確認できます：

{{< image link="./kiali-mtls.png" caption="Kiali ダッシュボード" >}}

詳細は[アプリとメトリクスの可視化](/ja/docs/ambient/getting-started/secure-and-visualize/#visualize-the-application-and-metrics)ドキュメントを参照してください。

## `tcpdump` で検証 {#validate-with-tcpdump}

Kubernetes ワーカーノードにアクセスできる場合、`tcpdump` コマンドでネットワークインターフェース上の全トラフィックをキャプチャできます。アプリケーションポートや HBONE ポートに絞ることも可能です。例では `9080` が `details` サービスのポート、`15008` が HBONE ポートです：

{{< text syntax=bash >}}
$ tcpdump -nAi eth0 port 9080 or port 15008
{{< /text >}}

`tcpdump` の出力で暗号化されたトラフィックが見えるはずです。

ワーカーノードにアクセスできない場合は、[netshoot コンテナイメージ](https://hub.docker.com/r/nicolaka/netshoot)を使って以下のコマンドを実行できます：

{{< text syntax=bash >}}
$ POD=$(kubectl get pods -l app=details -o jsonpath="{.items[0].metadata.name}")
$ kubectl debug $POD -i --image=nicolaka/netshoot -- tcpdump -nAi eth0 port 9080 or port 15008
{{< /text >}}
