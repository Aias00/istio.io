---
title: ztunnel の接続問題のトラブルシューティング
description: ノードプロキシが正しい設定を持っているかどうかを検証する方法。
weight: 60
owner: istio/wg-networking-maintainers
test: no
---

本ガイドでは、ztunnel プロキシの設定やデータパスを監視するためのいくつかのオプションを紹介します。
この情報は、高度なトラブルシューティングや、問題発生時にエラーレポートで収集・提供できる有用な情報の特定にも役立ちます。

## ztunnel プロキシの状態を確認する {#viewing-ztunnel-proxy-state}

ztunnel プロキシは xDS API を通じて istiod の {{< gloss "control plane" >}}コントロールプレーン{{< /gloss >}}から設定やディスカバリ情報を取得します。

`istioctl ztunnel-config` コマンドを使うと、ztunnel プロキシが認識しているワークロードを確認できます。

最初の例では、ztunnel が現在追跡しているすべてのワークロードやコントロールプレーンコンポーネントが表示されます。
これには、そのコンポーネントに接続する際に使用する IP アドレスやプロトコル、関連する waypoint プロキシの有無などの情報が含まれます。

{{< text bash >}}
$ istioctl ztunnel-config workloads
NAMESPACE POD NAME IP NODE WAYPOINT PROTOCOL
default bookinfo-gateway-istio-59dd7c96db-q9k6v 10.244.1.11 ambient-worker None TCP
default details-v1-cf74bb974-5sqkp 10.244.1.5 ambient-worker None HBONE
default productpage-v1-87d54dd59-fn6vw 10.244.1.10 ambient-worker None HBONE
default ratings-v1-7c4bbf97db-zvkdw 10.244.1.6 ambient-worker None HBONE
default reviews-v1-5fd6d4f8f8-knbht 10.244.1.16 ambient-worker None HBONE
default reviews-v2-6f9b55c5db-c94m2 10.244.1.17 ambient-worker None HBONE
default reviews-v3-7d99fd7978-7rgtd 10.244.1.18 ambient-worker None HBONE
default curl-7656cf8794-r7zb9 10.244.1.12 ambient-worker None HBONE
istio-system istiod-7ff4959459-qcpvp 10.244.2.5 ambient-worker2 None TCP
istio-system ztunnel-6hvcw 10.244.1.4 ambient-worker None TCP
istio-system ztunnel-mf476 10.244.2.6 ambient-worker2 None TCP
istio-system ztunnel-vqzf9 10.244.0.6 ambient-control-plane None TCP
kube-system coredns-76f75df574-2sms2 10.244.0.3 ambient-control-plane None TCP
kube-system coredns-76f75df574-5bf9c 10.244.0.2 ambient-control-plane None TCP
local-path-storage local-path-provisioner-7577fdbbfb-pslg6 10.244.0.4 ambient-control-plane None TCP

{{< /text >}}

`ztunnel-config` コマンドは、ztunnel プロキシが istiod コントロールプレーンから mTLS 用に受け取った証明書を保持している Secret も確認できます。

{{< text bash >}}
$ istioctl ztunnel-config certificates "$ZTUNNEL".istio-system
CERTIFICATE NAME TYPE STATUS VALID CERT SERIAL NUMBER NOT AFTER NOT BEFORE
spiffe://cluster.local/ns/default/sa/bookinfo-details Leaf Available true c198d859ee51556d0eae13b331b0c259 2024-05-05T09:17:47Z 2024-05-04T09:15:47Z
spiffe://cluster.local/ns/default/sa/bookinfo-details Root Available true bad086c516cce777645363cb8d731277 2034-04-24T03:31:05Z 2024-04-26T03:31:05Z
spiffe://cluster.local/ns/default/sa/bookinfo-productpage Leaf Available true 64c3828993c7df6f85a601a1615532cc 2024-05-05T09:17:47Z 2024-05-04T09:15:47Z
spiffe://cluster.local/ns/default/sa/bookinfo-productpage Root Available true bad086c516cce777645363cb8d731277 2034-04-24T03:31:05Z 2024-04-26T03:31:05Z
spiffe://cluster.local/ns/default/sa/bookinfo-ratings Leaf Available true 720479815bf6d81a05df8a64f384ebb0 2024-05-05T09:17:47Z 2024-05-04T09:15:47Z
spiffe://cluster.local/ns/default/sa/bookinfo-ratings Root Available true bad086c516cce777645363cb8d731277 2034-04-24T03:31:05Z 2024-04-26T03:31:05Z
spiffe://cluster.local/ns/default/sa/bookinfo-reviews Leaf Available true 285697fb2cf806852d3293298e300c86 2024-05-05T09:17:47Z 2024-05-04T09:15:47Z
spiffe://cluster.local/ns/default/sa/bookinfo-reviews Root Available true bad086c516cce777645363cb8d731277 2034-04-24T03:31:05Z 2024-04-26T03:31:05Z
spiffe://cluster.local/ns/default/sa/curl Leaf Available true fa33bbb783553a1704866842586e4c0b 2024-05-05T09:25:49Z 2024-05-04T09:23:49Z
spiffe://cluster.local/ns/default/sa/curl Root Available true bad086c516cce777645363cb8d731277 2034-04-24T03:31:05Z 2024-04-26T03:31:05Z
{{< /text >}}

これらのコマンドを使うことで、ztunnel プロキシがすべての想定ワークロードや TLS 証明書を正しく設定しているか確認できます。
また、情報が不足している場合はネットワークの問題の切り分けにも役立ちます。

`all` オプションを使うと、ztunnel-config のすべてのセクションを 1 つの CLI コマンドで確認できます：

{{< text bash >}}
$ istioctl ztunnel-config all -o json
{{< /text >}}

また、`curl` を使って ztunnel プロキシの生の設定ダンプを Pod 内のエンドポイントから取得することもできます：

{{< text bash >}}
$ kubectl debug -it $ZTUNNEL -n istio-system --image=curlimages/curl -- curl localhost:15000/config_dump
{{< /text >}}

## ztunnel xDS リソースの Istiod 状態を確認する {#viewing-istiod-state-for-ztunnel-xds-resources}

ztunnel プロキシ用に定義された xDS API リソースの形式で、
istiod コントロールプレーンが管理している ztunnel プロキシ設定リソースの状態を確認したい場合があります。
これは、istiod Pod に入り、指定した ztunnel プロキシの 15014 ポートから情報を取得することで可能です。
また、出力を JSON 整形ツールで保存・閲覧することもできます（例では省略）。

{{< text bash >}}
$ export ISTIOD=$(kubectl get pods -n istio-system -l app=istiod -o=jsonpath='{.items[0].metadata.name}')
$ kubectl debug -it $ISTIOD -n istio-system --image=curlimages/curl -- curl localhost:15014/debug/config_dump?proxyID="$ZTUNNEL".istio-system
{{< /text >}}

## ログで ztunnel トラフィックを検証する {#verifying-ztunnel-traffic-through-logs}

標準的な Kubernetes ログツールを使って、ztunnel のトラフィックログを確認できます。

{{< text bash >}}
$ kubectl -n default exec deploy/curl -- sh -c 'for i in $(seq 1 10); do curl -s -I http://productpage:9080/; done'
HTTP/1.1 200 OK
Server: Werkzeug/3.0.1 Python/3.12.1
--snip--
{{< /text >}}

レスポンスは、クライアント Pod がサービスから応答を受け取ったことを示します。
次に、ztunnel Pod のログを確認し、トラフィックが HBONE トンネル経由で送信されたことを確認します。

{{< text bash >}}
$ kubectl -n istio-system logs -l app=ztunnel | grep -E "inbound|outbound"
2024-05-04T09:59:05.028709Z info access connection complete src.addr=10.244.1.12:60059 src.workload="curl-7656cf8794-r7zb9" src.namespace="default" src.identity="spiffe://cluster.local/ns/default/sa/curl" dst.addr=10.244.1.10:9080 dst.hbone_addr="10.244.1.10:9080" dst.service="productpage.default.svc.cluster.local" dst.workload="productpage-v1-87d54dd59-fn6vw" dst.namespace="productpage" dst.identity="spiffe://cluster.local/ns/default/sa/bookinfo-productpage" direction="inbound" bytes_sent=175 bytes_recv=80 duration="1ms"
2024-05-04T09:59:05.028771Z info access connection complete src.addr=10.244.1.12:58508 src.workload="curl-7656cf8794-r7zb9" src.namespace="default" src.identity="spiffe://cluster.local/ns/default/sa/curl" dst.addr=10.244.1.10:15008 dst.hbone_addr="10.244.1.10:9080" dst.service="productpage.default.svc.cluster.local" dst.workload="productpage-v1-87d54dd59-fn6vw" dst.namespace="productpage" dst.identity="spiffe://cluster.local/ns/default/sa/bookinfo-productpage" direction="outbound" bytes_sent=80 bytes_recv=175 duration="1ms"
--snip--
{{< /text >}}

これらのログメッセージは、トラフィックが ztunnel プロキシ経由で送信されたことを確認するものです。
トラフィックの送信元 Pod と宛先 Pod が同じノード上にある場合は、そのノード上の ztunnel プロキシインスタンスのログを確認することで、さらに詳細な監視が可能です。
これらのログが見つからない場合は、[トラフィックリダイレクト](/ja/docs/ambient/architecture/traffic-redirection)が正しく動作していない可能性があります。

{{< tip >}}
トラフィックの送信元と宛先が同じ計算ノード上にあっても、トラフィックは必ず ztunnel Pod を経由します。
{{< /tip >}}

### ztunnel の負荷分散の検証 {#verifying-ztunnel-load-balancing}

ターゲットが複数のエンドポイントを持つサービスの場合、ztunnel プロキシは自動的にクライアント負荷分散を行います。
追加の設定は不要です。負荷分散アルゴリズムは内部で固定された L4 ラウンドロビンで、L4 コネクションの状態に基づいてトラフィックを分配します。ユーザーによる設定はできません。

{{< tip >}}
ターゲットが複数のインスタンスや Pod を持つサービスで、ターゲットサービスに waypoint が関連付けられていない場合、
送信元 ztunnel はそれらのインスタンスやサービスバックエンド間で直接 L4 負荷分散を行い、
それぞれのバックエンドに関連付けられたリモート ztunnel プロキシ経由でトラフィックを送信します。
ターゲットサービスが 1 つ以上の waypoint プロキシを使うよう設定されている場合、
送信元 ztunnel プロキシは waypoint プロキシ間でトラフィックを分配し、
waypoint プロキシインスタンスが存在するノード上のリモート ztunnel プロキシ経由でトラフィックを送信します。
{{< /tip >}}

複数のバックエンドを持つサービスにリクエストを送信することで、クライアントトラフィックがサービスレプリカ間で分散されているか検証できます。

{{< text bash >}}
$ kubectl -n default exec deploy/curl -- sh -c 'for i in $(seq 1 10); do curl -s -I http://reviews:9080/; done'
{{< /text >}}

{{< text bash >}}
$ kubectl -n istio-system logs -l app=ztunnel | grep -E "outbound"
--snip--
2024-05-04T10:11:04.964851Z info access connection complete src.addr=10.244.1.12:35520 src.workload="curl-7656cf8794-r7zb9" src.namespace="default" src.identity="spiffe://cluster.local/ns/default/sa/curl" dst.addr=10.244.1.9:15008 dst.hbone_addr="10.244.1.9:9080" dst.service="reviews.default.svc.cluster.local" dst.workload="reviews-v3-7d99fd7978-zznnq" dst.namespace="reviews" dst.identity="spiffe://cluster.local/ns/default/sa/bookinfo-reviews" direction="outbound" bytes_sent=84 bytes_recv=169 duration="2ms"
2024-05-04T10:11:04.969578Z info access connection complete src.addr=10.244.1.12:35526 src.workload="curl-7656cf8794-r7zb9" src.namespace="default" src.identity="spiffe://cluster.local/ns/default/sa/curl" dst.addr=10.244.1.9:15008 dst.hbone_addr="10.244.1.9:9080" dst.service="reviews.default.svc.cluster.local" dst.workload="reviews-v3-7d99fd7978-zznnq" dst.namespace="reviews" dst.identity="spiffe://cluster.local/ns/default/sa/bookinfo-reviews" direction="outbound" bytes_sent=84 bytes_recv=169 duration="2ms"
2024-05-04T10:11:04.974720Z info access connection complete src.addr=10.244.1.12:35536 src.workload="curl-7656cf8794-r7zb9" src.namespace="default" src.identity="spiffe://cluster.local/ns/default/sa/curl" dst.addr=10.244.1.7:15008 dst.hbone_addr="10.244.1.7:9080" dst.service="reviews.default.svc.cluster.local" dst.workload="reviews-v1-5fd6d4f8f8-26j92" dst.namespace="reviews" dst.identity="spiffe://cluster.local/ns/default/sa/bookinfo-reviews" direction="outbound" bytes_sent=84 bytes_recv=169 duration="2ms"
2024-05-04T10:11:04.979462Z info access connection complete src.addr=10.244.1.12:35552 src.workload="curl-7656cf8794-r7zb9" src.namespace="default" src.identity="spiffe://cluster.local/ns/default/sa/curl" dst.addr=10.244.1.8:15008 dst.hbone_addr="10.244.1.8:9080" dst.service="reviews.default.svc.cluster.local" dst.workload="reviews-v2-6f9b55c5db-c2dtw" dst.namespace="reviews" dst.identity="spiffe://cluster.local/ns/default/sa/bookinfo-reviews" direction="outbound" bytes_sent=84 bytes_recv=169 duration="2ms"
{{< /text >}}

これはラウンドロビン型の負荷分散アルゴリズムであり、
`VirtualService` の `TrafficPolicy` フィールドで設定できる他の負荷分散アルゴリズムとは独立しています。
（前述の通り、`VirtualService` API オブジェクトのすべての側面は ztunnel ではなく waypoint プロキシ上でインスタンス化されます）。

### Ambient モードトラフィックの可観測性 {#observability-of-ambient-mode-traffic}

ztunnel のログや上記の監視オプションに加え、
通常の Istio の監視・可観測性機能を使って Ambient データプレーンモードのアプリケーショントラフィックを監視できます。

- [Prometheus のインストール](/ja/docs/ops/integrations/prometheus/#installation)
- [Kiali のインストール](/ja/docs/ops/integrations/kiali/#installation)
- [Istio メトリクス](/ja/docs/reference/config/metrics/)
- [Prometheus からのメトリクスクエリ](/ja/docs/tasks/observability/metrics/querying-metrics/)

サービスが ztunnel のセキュアカバレッジのみを利用している場合、報告される Istio メトリクスは L4 TCP メトリクス（`istio_tcp_sent_bytes_total`、`istio_tcp_received_bytes_total`、`istio_tcp_connections_opened_total`、`istio_tcp_connections_filled_total`）のみです。
waypoint プロキシを利用している場合は、Istio および Envoy の全メトリクスが報告されます。
