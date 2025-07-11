---
title: サーキットブレーカー
description: このタスクでは、接続、リクエスト、および異常検出のためのサーキットブレーカーの設定方法を紹介します。
weight: 50
keywords: [traffic-management, circuit-breaking]
owner: istio/wg-networking-maintainers
test: yes
---

このタスクでは、接続、リクエスト、および異常検出のためのサーキットブレーカーの設定方法を紹介します。

サーキットブレーカーは、レジリエントなマイクロサービスアプリケーションを構築するための重要なパターンです。
サーキットブレーカーを使うことで、アプリケーションは障害、突発的なピーク、その他の未知のネットワーク要因に対して耐性を持つことができます。

このタスクでは、サーキットブレーカーのルールを設定し、意図的にサーキットブレーカーを「トリップ」させて設定をテストします。

## 始める前に {#before-you-begin}

- [インストールガイド](/ja/docs/setup/) に従って Istio をインストールしてください。

{{< boilerplate start-httpbin-service >}}

アプリケーション `httpbin` はこのタスクのバックエンドサービスとして使用されます。

## サーキットブレーカーの設定 {#configuring-the-circuit-breaker}

1. [DestinationRule](/ja/docs/reference/config/networking/destination-rule/) を作成し、
   `httpbin` サービスへの呼び出し時にサーキットブレーカー設定を適用します：

   {{< warning >}}
   Istio で双方向 TLS 認証が有効な場合は、DestinationRule を適用する前に TLS トラフィックポリシー
   `mode: ISTIO_MUTUAL` を追加する必要があります。そうしないと、リクエストが 503 エラーになります。
   詳細は[こちら](/ja/docs/ops/common-problems/network-issues/#service-unavailable-errors-after-setting-destination-rule)を参照してください。
   {{< /warning >}}

   {{< text bash >}}
   $ kubectl apply -f - <<EOF
   apiVersion: networking.istio.io/v1
   kind: DestinationRule
   metadata:
   name: httpbin
   spec:
   host: httpbin
   trafficPolicy:
   connectionPool:
   tcp:
   maxConnections: 1
   http:
   http1MaxPendingRequests: 1
   maxRequestsPerConnection: 1
   outlierDetection:
   consecutive5xxErrors: 1
   interval: 1s
   baseEjectionTime: 3m
   maxEjectionPercent: 100
   EOF
   {{< /text >}}

1. DestinationRule が正しく作成されたか確認します：

   {{< text bash yaml >}}
   $ kubectl get destinationrule httpbin -o yaml
   apiVersion: networking.istio.io/v1
   kind: DestinationRule
   ...
   spec:
   host: httpbin
   trafficPolicy:
   connectionPool:
   http:
   http1MaxPendingRequests: 1
   maxRequestsPerConnection: 1
   tcp:
   maxConnections: 1
   outlierDetection:
   baseEjectionTime: 3m
   consecutive5xxErrors: 1
   interval: 1s
   maxEjectionPercent: 100
   {{< /text >}}

## クライアントの追加 {#adding-a-client}

`httpbin` サービスにトラフィックを送信するクライアントプログラムを作成します。これは [Fortio](https://github.com/istio/fortio) という負荷テストクライアントで、
接続数、並列数、HTTP リクエストの遅延を制御できます。
Fortio を使うことで、`DestinationRule` で設定したサーキットブレーカーの動作を効果的にトリガーできます。

1. クライアントに Istio Sidecar プロキシを注入し、Istio がそのネットワーク通信を管理できるようにします：

   [Sidecar の自動注入](/ja/docs/setup/additional-setup/sidecar-injection/#automatic-sidecar-injection)が有効な場合は、
   `fortio` アプリをそのままデプロイできます：

   {{< text bash >}}
   $ kubectl apply -f @samples/httpbin/sample-client/fortio-deploy.yaml@
   {{< /text >}}

   そうでない場合は、`fortio` アプリをデプロイする前に手動で Sidecar を注入してください：

   {{< text bash >}}
   $ kubectl apply -f <(istioctl kube-inject -f @samples/httpbin/sample-client/fortio-deploy.yaml@)
   {{< /text >}}

1. クライアント Pod に入り、Fortio ツールで `httpbin` サービスを呼び出します。`-curl` オプションは 1 回だけ呼び出すことを意味します：

   {{< text bash >}}
   $ export FORTIO_POD=$(kubectl get pods -l app=fortio -o 'jsonpath={.items[0].metadata.name}')
    $ kubectl exec "$FORTIO_POD" -c fortio -- /usr/bin/fortio curl -quiet http://httpbin:8000/get
   HTTP/1.1 200 OK
   server: envoy
   date: Tue, 25 Feb 2020 20:25:52 GMT
   content-type: application/json
   content-length: 586
   access-control-allow-origin: \*
   access-control-allow-credentials: true
   x-envoy-upstream-service-time: 36

   {
   "args": {},
   "headers": {
   "Content-Length": "0",
   "Host": "httpbin:8000",
   "User-Agent": "fortio.org/fortio-1.3.1",
   "X-B3-Parentspanid": "8fc453fb1dec2c22",
   "X-B3-Sampled": "1",
   "X-B3-Spanid": "071d7f06bc94943c",
   "X-B3-Traceid": "86a929a0e76cda378fc453fb1dec2c22",
   "X-Forwarded-Client-Cert": "By=spiffe://cluster.local/ns/default/sa/httpbin;Hash=68bbaedefe01ef4cb99e17358ff63e92d04a4ce831a35ab9a31d3c8e06adb038;Subject=\"\";URI=spiffe://cluster.local/ns/default/sa/default"
   },
   "origin": "127.0.0.1",
   "url": "http://httpbin:8000/get"
   }
   {{< /text >}}

リクエストがバックエンドサービスに正常に到達したことが確認できます。次に、サーキットブレーカーのテストを行います。

## サーキットブレーカーをトリップさせる {#tripping-the-circuit-breaker}

`DestinationRule` の設定で `maxConnections: 1` と `http1MaxPendingRequests: 1` を指定しました。
これらのルールは、同時接続数とリクエスト数が 1 を超えると、`istio-proxy` でそれ以上のリクエストや接続がブロックされることを意味します。

1. 並列数 2（`-c 2`）、リクエスト数 20（`-n 20`）でリクエストを送信します：

   {{< text bash >}}
   $ kubectl exec "$FORTIO_POD" -c fortio -- /usr/bin/fortio load -c 2 -qps 0 -n 20 -loglevel Warning http://httpbin:8000/get
   ...
   Code 200 : 17 (85.0 %)
   Code 503 : 3 (15.0 %)
   {{< /text >}}

   ほとんどのリクエストが完了していますが、`istio-proxy` は多少の誤差を許容します。

   {{< text plain >}}
   Code 200 : 17 (85.0 %)
   Code 503 : 3 (15.0 %)
   {{< /text >}}

1. 並列接続数を 3 に増やします：

   {{< text bash >}}
   $ kubectl exec "$FORTIO_POD" -c fortio -- /usr/bin/fortio load -c 3 -qps 0 -n 30 -loglevel Warning http://httpbin:8000/get
   ...
   Code 200 : 11 (36.7 %)
   Code 503 : 19 (63.3 %)
   {{< /text >}}

   今度は、期待通りのサーキットブレーカー動作が見られ、36.7% のリクエストのみが成功し、残りはサーキットブレーカーによってブロックされます：

   {{< text plain >}}
   Code 200 : 11 (36.7 %)
   Code 503 : 19 (63.3 %)
   {{< /text >}}

1. `istio-proxy` の状態を確認し、サーキットブレーカーの詳細を調べます：

   {{< text bash >}}
   $ kubectl exec "$FORTIO_POD" -c istio-proxy -- pilot-agent request GET stats | grep httpbin | grep pending
   cluster.outbound|8000||httpbin.default.svc.cluster.local;.circuit_breakers.default.remaining_pending: 1
   cluster.outbound|8000||httpbin.default.svc.cluster.local;.circuit_breakers.default.rq_pending_open: 0
   cluster.outbound|8000||httpbin.default.svc.cluster.local;.circuit_breakers.high.rq_pending_open: 0
   cluster.outbound|8000||httpbin.default.svc.cluster.local;.upstream_rq_pending_active: 0
   cluster.outbound|8000||httpbin.default.svc.cluster.local;.upstream_rq_pending_failure_eject: 0
   cluster.outbound|8000||httpbin.default.svc.cluster.local;.upstream_rq_pending_overflow: 21
   cluster.outbound|8000||httpbin.default.svc.cluster.local;.upstream_rq_pending_total: 29
   {{< /text >}}

   `upstream_rq_pending_overflow` の値が `21` となっていることが分かります。これは、これまでに 21 件のリクエストがサーキットブレーカーによってブロックされたことを意味します。

## クリーンアップ {#cleaning-up}

1. ルールのクリーンアップ：

   {{< text bash >}}
   $ kubectl delete destinationrule httpbin
   {{< /text >}}

1. [httpbin]({{< github_tree >}}/samples/httpbin) サービスとクライアントを削除します：

   {{< text bash >}}
   $ kubectl delete -f @samples/httpbin/sample-client/fortio-deploy.yaml@
   $ kubectl delete -f @samples/httpbin/httpbin.yaml@
   {{< /text >}}
