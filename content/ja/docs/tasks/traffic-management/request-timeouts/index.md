---
title: リクエストタイムアウトの設定
description: このタスクでは Istio を使って Envoy でリクエストタイムアウトを設定する方法を紹介します。
weight: 40
aliases:
  - /zh/docs/tasks/request-timeouts.html
keywords: [traffic-management, timeouts]
owner: istio/wg-networking-maintainers
test: yes
---

このタスクでは、Istio を使って Envoy でリクエストタイムアウトを設定する方法を紹介します。

{{< boilerplate gateway-api-support >}}

## 始める前に {#before-you-begin}

- [インストールガイド](/ja/docs/setup/) の手順に従って Istio をインストールします。

- サンプルアプリケーション [Bookinfo](/ja/docs/examples/bookinfo/) をデプロイし、[サービスバージョン](/ja/docs/examples/bookinfo/#define-the-service-versions) も含めてセットアップします。

## リクエストタイムアウト {#request-timeouts}

HTTP リクエストのタイムアウトはルーティングルールの timeout フィールドで指定できます。
デフォルトではタイムアウトは無効です。このタスクでは `reviews` サービスのタイムアウトを 0.5 秒に設定します。
効果を観察するため、`ratings` サービスへの呼び出しに 2 秒の遅延を人工的に挿入します。

1. リクエストを `reviews` サービスの v2 バージョンにルーティングします。これは `ratings` サービスへの呼び出しを行います：

{{< tabset category-name="config-api" >}}

{{< tab name="Istio APIs" category-value="istio-apis" >}}

{{< text bash >}}
$ kubectl apply -f - <<EOF
apiVersion: networking.istio.io/v1
kind: VirtualService
metadata:
name: reviews
spec:
hosts: - reviews
http:

- route: - destination:
  host: reviews
  subset: v2
  EOF
  {{< /text >}}

{{< /tab >}}

{{< tab name="Gateway API" category-value="gateway-api" >}}

{{< text bash >}}
$ kubectl apply -f - <<EOF
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
name: reviews
spec:
parentRefs:

- group: ""
  kind: Service
  name: reviews
  port: 9080
  rules:
- backendRefs: - name: reviews-v2
  port: 9080
  EOF
  {{< /text >}}

{{< /tab >}}

{{< /tabset >}}

2. `ratings` サービスへの呼び出しに 2 秒の遅延を追加します：

{{< tabset category-name="config-api" >}}

{{< tab name="Istio APIs" category-value="istio-apis" >}}

{{< text bash >}}
$ kubectl apply -f - <<EOF
apiVersion: networking.istio.io/v1
kind: VirtualService
metadata:
name: ratings
spec:
hosts:

- ratings
  http:
- fault:
  delay:
  percentage:
  value: 100
  fixedDelay: 2s
  route: - destination:
  host: ratings
  subset: v1
  EOF
  {{< /text >}}

{{< /tab >}}

{{< tab name="Gateway API" category-value="gateway-api" >}}

Gateway API では故障注入はまだサポートされていないため、
遅延を追加するには Istio の `VirtualService` を使います：

{{< text bash >}}
$ kubectl apply -f - <<EOF
apiVersion: networking.istio.io/v1
kind: VirtualService
metadata:
name: ratings
spec:
hosts:

- ratings
  http:
- fault:
  delay:
  percentage:
  value: 100
  fixedDelay: 2s
  route: - destination:
  host: ratings
  EOF
  {{< /text >}}

{{< /tab >}}

{{< /tabset >}}

3. ブラウザで Bookinfo の URL `http://$GATEWAY_URL/productpage` を開きます。
   `$GATEWAY_URL` は外部 Ingress IP アドレスです（[Bookinfo](/ja/docs/examples/bookinfo/#determine-the-ingress-ip-and-port) ドキュメント参照）。

   これで Bookinfo アプリが正常に動作していることが確認できます（星型の評価が表示されます）が、ページをリロードするたびに 2 秒の遅延が発生します。

4. `reviews` サービスへの呼び出しに 0.5 秒のリクエストタイムアウトを追加します：

{{< tabset category-name="config-api" >}}

{{< tab name="Istio APIs" category-value="istio-apis" >}}

{{< text bash >}}
$ kubectl apply -f - <<EOF
apiVersion: networking.istio.io/v1
kind: VirtualService
metadata:
name: reviews
spec:
hosts:

- reviews
  http:
- route: - destination:
  host: reviews
  subset: v2
  timeout: 0.5s
  EOF
  {{< /text >}}

{{< /tab >}}

{{< tab name="Gateway API" category-value="gateway-api" >}}

{{< text bash >}}
$ kubectl apply -f - <<EOF
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
name: reviews
spec:
parentRefs:

- group: ""
  kind: Service
  name: reviews
  port: 9080
  rules:
- backendRefs: - name: reviews-v2
  port: 9080
  timeouts:
  request: 500ms
  EOF
  {{< /text >}}

{{< /tab >}}

{{< /tabset >}}

5. Bookinfo ページをリロードします。

   今度は 1 秒ほどでレスポンスが返るはずですが、`reviews` は利用できません。

   {{< tip >}}
   タイムアウトを 0.5 秒に設定しても、レスポンスが 1 秒かかるのは、`productpage` サービスにハードコードされたリトライがあるためです。
   そのため、返る前に `reviews` サービスのタイムアウトが 2 回発生します。
   {{< /tip >}}

## 仕組みの理解 {#understanding-what-happened}

このタスクでは、Istio を使って `reviews` マイクロサービスへの呼び出しに 0.5 秒のリクエストタイムアウトを設定しました。デフォルトではリクエストタイムアウトは無効です。
`reviews` サービスはリクエスト処理時に `ratings` サービスを呼び出しますが、Istio で `ratings` への呼び出しに 2 秒の遅延を注入したため、`reviews` サービスは 0.5 秒以内に完了できず、タイムアウトが発生します。

Bookinfo のページ（`reviews` サービスを呼び出して生成される）は、レビューが表示されず、次のメッセージが表示されます：
**Sorry, product reviews are currently unavailable for this book.**
これは `reviews` サービスからタイムアウトエラーが返されたためです。

[故障注入タスク](/ja/docs/tasks/traffic-management/fault-injection/) を見たことがある場合、`productpage` マイクロサービスが `reviews` マイクロサービスを呼び出す際に、アプリケーションレベルで 3 秒のタイムアウトを設定していることがわかります。
このタスクでは Istio のルーティングルールで 0.5 秒のタイムアウトを設定しました。もしタイムアウトを 3 秒より大きく（例：4 秒）設定した場合、タイムアウトは発生しません。なぜなら、より厳しい方のタイムアウトが優先されるためです。詳細は [こちら](/ja/docs/concepts/traffic-management/#network-resilience-and-testing) を参照してください。

Istio のタイムアウト制御について補足すると、この記事のようにルーティングルールでタイムアウトを設定する以外に、リクエスト単位で `x-envoy-upstream-rq-timeout-ms` リクエストヘッダーを付与することでも設定できます。このヘッダーの値はミリ秒単位です。

## クリーンアップ {#cleanup}

- アプリケーションのルーティングルールを削除します：

{{< tabset category-name="config-api" >}}

{{< tab name="Istio APIs" category-value="istio-apis" >}}

{{< text bash >}}
$ kubectl delete -f @samples/bookinfo/networking/virtual-service-all-v1.yaml@
{{< /text >}}

{{< /tab >}}

{{< tab name="Gateway API" category-value="gateway-api" >}}

{{< text bash >}}
$ kubectl delete httproute reviews
$ kubectl delete virtualservice ratings
{{< /text >}}

{{< /tab >}}

{{< /tabset >}}

- 今後のタスクを試す予定がない場合は、[Bookinfo のクリーンアップ](/ja/docs/examples/bookinfo/#cleanup) の手順に従ってアプリケーションを停止してください。
