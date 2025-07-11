---
title: ミラーリング
description: このタスクでは Istio のトラフィックミラーリング/シャドー機能を紹介します。
weight: 60
keywords: [traffic-management, mirroring]
owner: istio/wg-networking-maintainers
test: yes
---

このタスクでは Istio のトラフィックミラーリング機能を紹介します。

トラフィックミラーリング（シャドートラフィックとも呼ばれる）は、責任ある機能チームが本番環境にできるだけリスクを抑えて変更を加えるための強力な手法です。
ミラーリングはリアルタイムトラフィックのコピーをミラーサービスに送信します。ミラーリングトラフィックはメインサービスのクリティカルなリクエストパスの外側で発生します。

このタスクでは、まずすべてのトラフィックをテストサービスの `v1` バージョンにルーティングします。その後、一部のトラフィックを `v2` バージョンにミラーリングするルールを適用します。

{{< boilerplate gateway-api-support >}}

## 始める前に {#before-you-begin}

1. [インストールガイド](/ja/docs/setup/) に従って Istio をセットアップします。
1. アクセスログ記録が有効な 2 つのバージョンの [httpbin]({{< github_tree >}}/samples/httpbin) サービスをデプロイします：

   1. `httpbin-v1` のデプロイ：

      {{< text bash >}}
      $ kubectl create -f - <<EOF
      apiVersion: apps/v1
      kind: Deployment
      metadata:
      name: httpbin-v1
      spec:
      replicas: 1
      selector:
      matchLabels:
      app: httpbin
      version: v1
      template:
      metadata:
      labels:
      app: httpbin
      version: v1
      spec:
      containers: - image: docker.io/kennethreitz/httpbin
      imagePullPolicy: IfNotPresent
      name: httpbin
      command: ["gunicorn", "--access-logfile", "-", "-b", "0.0.0.0:80", "httpbin:app"]
      ports: - containerPort: 80
      EOF
      {{< /text >}}

   1. `httpbin-v2` のデプロイ：

      {{< text bash >}}
      $ kubectl create -f - <<EOF
      apiVersion: apps/v1
      kind: Deployment
      metadata:
      name: httpbin-v2
      spec:
      replicas: 1
      selector:
      matchLabels:
      app: httpbin
      version: v2
      template:
      metadata:
      labels:
      app: httpbin
      version: v2
      spec:
      containers: - image: docker.io/kennethreitz/httpbin
      imagePullPolicy: IfNotPresent
      name: httpbin
      command: ["gunicorn", "--access-logfile", "-", "-b", "0.0.0.0:80", "httpbin:app"]
      ports: - containerPort: 80
      EOF
      {{< /text >}}

   1. `httpbin` Kubernetes Service のデプロイ：

      {{< text bash >}}
      $ kubectl create -f - <<EOF
      apiVersion: v1
      kind: Service
      metadata:
      name: httpbin
      labels:
      app: httpbin
      spec:
      ports:

      - name: http
        port: 8000
        targetPort: 80
        selector:
        app: httpbin
        EOF
        {{< /text >}}

1. `httpbin` サービスにリクエストを送信するための `curl` ワークロードをデプロイします：

   {{< text bash >}}
   $ cat <<EOF | kubectl create -f -
   apiVersion: apps/v1
   kind: Deployment
   metadata:
   name: curl
   spec:
   replicas: 1
   selector:
   matchLabels:
   app: curl
   template:
   metadata:
   labels:
   app: curl
   spec:
   containers: - name: curl
   image: curlimages/curl
   command: ["/bin/sleep","3650d"]
   imagePullPolicy: IfNotPresent
   EOF
   {{< /text >}}

## デフォルトルーティングポリシーの作成 {#creating-a-default-routing-policy}

デフォルトでは、Kubernetes は `httpbin` サービスの 2 つのバージョン間で負荷分散します。
このステップでは、すべてのトラフィックを `v1` バージョンにルーティングするように動作を変更します。

1. すべてのトラフィックをサービスの `v1` バージョンにルーティングするデフォルトルールを作成します：

   {{< tabset category-name="config-api" >}}

   {{< tab name="Istio API" category-value="istio-apis" >}}

   {{< text bash >}}
   $ kubectl apply -f - <<EOF
   apiVersion: networking.istio.io/v1
   kind: VirtualService
   metadata:
   name: httpbin
   spec:
   hosts: - httpbin
   http:

   - route:
     - destination:
       host: httpbin
       subset: v1
       weight: 100

   ***

   apiVersion: networking.istio.io/v1
   kind: DestinationRule
   metadata:
   name: httpbin
   spec:
   host: httpbin
   subsets:

   - name: v1
     labels:
     version: v1
   - name: v2
     labels:
     version: v2
     EOF
     {{< /text >}}

   {{< /tab >}}

   {{< tab name="Gateway API" category-value="gateway-api" >}}

   {{< text bash >}}
   $ kubectl apply -f - <<EOF
   apiVersion: v1
   kind: Service
   metadata:
   name: httpbin-v1
   spec:
   ports:

   - port: 80
     name: http
     selector:
     app: httpbin
     version: v1

   ***

   apiVersion: v1
   kind: Service
   metadata:
   name: httpbin-v2
   spec:
   ports:

   - port: 80
     name: http
     selector:
     app: httpbin
     version: v2

   ***

   apiVersion: gateway.networking.k8s.io/v1
   kind: HTTPRoute
   metadata:
   name: httpbin
   spec:
   parentRefs:

   - group: ""
     kind: Service
     name: httpbin
     port: 8000
     rules:
   - backendRefs: - name: httpbin-v1
     port: 80
     EOF
     {{< /text >}}

   {{< /tab >}}

   {{< /tabset >}}

1. これで全トラフィックが `httpbin:v1` サービスに向かうようになり、リクエストを送信できます：

   {{< text bash json >}}
   $ kubectl exec deploy/curl -c curl -- curl -sS http://httpbin:8000/headers
   {
   "headers": {
   "Accept": "_/_",
   "Content-Length": "0",
   "Host": "httpbin:8000",
   "User-Agent": "curl/7.35.0",
   "X-B3-Parentspanid": "57784f8bff90ae0b",
   "X-B3-Sampled": "1",
   "X-B3-Spanid": "3289ae7257c3f159",
   "X-B3-Traceid": "b56eebd279a76f0b57784f8bff90ae0b",
   "X-Envoy-Attempt-Count": "1",
   "X-Forwarded-Client-Cert": "By=spiffe://cluster.local/ns/default/sa/default;Hash=20afebed6da091c850264cc751b8c9306abac02993f80bdb76282237422bd098;Subject=\"\";URI=spiffe://cluster.local/ns/default/sa/default"
   }
   }
   {{< /text >}}

1. `httpbin-v1` と `httpbin-v2` の 2 つの Pod のログを確認します。
   `v1` バージョンのアクセスログエントリが見え、`v2` にはログがないはずです：

   {{< text bash >}}
   $ kubectl logs deploy/httpbin-v1 -c httpbin
   127.0.0.1 - - [07/Mar/2018:19:02:43 +0000] "GET /headers HTTP/1.1" 200 321 "-" "curl/7.35.0"
   {{< /text >}}

   {{< text bash >}}
   $ kubectl logs deploy/httpbin-v2 -c httpbin
   <none>
   {{< /text >}}

## トラフィックを `httpbin-v2` にミラーリングする {#mirroring-traffic-to-httpbin-v2}

1. ルールを変更してトラフィックを `httpbin-v2` にミラーリングします：

   {{< tabset category-name="config-api" >}}

   {{< tab name="Istio API" category-value="istio-apis" >}}

   {{< text bash >}}
   $ kubectl apply -f - <<EOF
   apiVersion: networking.istio.io/v1
   kind: VirtualService
   metadata:
   name: httpbin
   spec:
   hosts: - httpbin
   http:

   - route: - destination:
     host: httpbin
     subset: v1
     weight: 100
     mirror:
     host: httpbin
     subset: v2
     mirrorPercentage:
     value: 100.0
     EOF
     {{< /text >}}

   このルールは 100% のトラフィックを `v1` バージョンに送ります。
   最後のセクションで、同じトラフィックの 100% をミラー（つまり `httpbin:v2` にも送信）しています。
   ミラーリング時、リクエストはミラーサービスに送信され、`headers` の `Host/Authority` 属性値に `-shadow` が追加されます。
   例：`cluster-1` → `cluster-1-shadow`。

   また、ミラーされたトラフィックは「投げっぱなし」であり、ミラーリクエストのレスポンスは破棄されます。

   `mirrorPercentage` の `value` フィールドでミラーリングするトラフィックの割合を指定できます。
   この属性がなければ全トラフィックがミラーされます。

   {{< /tab >}}

   {{< tab name="Gateway API" category-value="gateway-api" >}}

   {{< text bash >}}
   $ kubectl apply -f - <<EOF
   apiVersion: gateway.networking.k8s.io/v1
   kind: HTTPRoute
   metadata:
   name: httpbin
   spec:
   parentRefs:

   - group: ""
     kind: Service
     name: httpbin
     port: 8000
     rules:
   - filters: - type: RequestMirror
     requestMirror:
     backendRef:
     name: httpbin-v2
     port: 80
     backendRefs: - name: httpbin-v1
     port: 80
     EOF
     {{< /text >}}

   このルールは 100% のトラフィックを `v1` に送ります。
   `RequestMirror` フィルターで、同じトラフィックの 100% を `httpbin:v2` にミラーしています。
   ミラーリング時、リクエストはミラーサービスに送信され、Host/Authority ヘッダーに `-shadow` が追加されます。
   例：`cluster-1` → `cluster-1-shadow`。

   また、ミラーされたトラフィックは「投げっぱなし」であり、ミラーリクエストのレスポンスは破棄されます。

   {{< /tab >}}

   {{< /tabset >}}

1. トラフィックを送信します：

   {{< text bash >}}
   $ kubectl exec deploy/curl -c curl -- curl -sS http://httpbin:8000/headers
   {{< /text >}}

   これで `v1` と `v2` の両方のバージョンでアクセスログが見えるはずです。
   `v2` のアクセスログはミラーリングトラフィックによるもので、実際のリクエストのターゲットは `v1` です。

   {{< text bash >}}
   $ kubectl logs deploy/httpbin-v1 -c httpbin
   127.0.0.1 - - [07/Mar/2018:19:02:43 +0000] "GET /headers HTTP/1.1" 200 321 "-" "curl/7.35.0"
   127.0.0.1 - - [07/Mar/2018:19:26:44 +0000] "GET /headers HTTP/1.1" 200 321 "-" "curl/7.35.0"
   {{< /text >}}

   {{< text bash >}}
   $ kubectl logs deploy/httpbin-v2 -c httpbin
   127.0.0.1 - - [07/Mar/2018:19:26:44 +0000] "GET /headers HTTP/1.1" 200 361 "-" "curl/7.35.0"
   {{< /text >}}

## クリーンアップ {#cleaning-up}

1. ルールを削除します：

   {{< tabset category-name="config-api" >}}

   {{< tab name="Istio API" category-value="istio-apis" >}}

   {{< text bash >}}
   $ kubectl delete virtualservice httpbin
   $ kubectl delete destinationrule httpbin
   {{< /text >}}

   {{< /tab >}}

   {{< tab name="Gateway API" category-value="gateway-api" >}}

   {{< text bash >}}
   $ kubectl delete httproute httpbin
   $ kubectl delete svc httpbin-v1 httpbin-v2
   {{< /text >}}

   {{< /tab >}}

   {{< /tabset >}}

1. `httpbin` と `curl` の Deployment および `httpbin` サービスを削除します：

   {{< text bash >}}
   $ kubectl delete deploy httpbin-v1 httpbin-v2 curl
   $ kubectl delete svc httpbin
   {{< /text >}}
