---
title: Ingress ゲートウェイ
description: Istio Gateway オブジェクトを使ってサービスをサービスメッシュ外部に公開する方法を説明します。
weight: 10
keywords: [traffic-management, ingress]
aliases:
  - /zh/docs/tasks/ingress.html
  - /zh/docs/tasks/ingress
owner: istio/wg-networking-maintainers
test: yes
---

Kubernetes [Ingress](/ja/docs/tasks/traffic-management/ingress/kubernetes-ingress/) のサポートに加え、
Istio では [Istio Gateway](/ja/docs/concepts/traffic-management/#gateways) や [Kubernetes Gateway](https://gateway-api.sigs.k8s.io/api-types/gateway/) リソースを使って Ingress トラフィックを構成できます。
`Ingress` と比べて `Gateway` はより柔軟でカスタマイズ性が高く、Istio の機能（監視やルーティングルールなど）をクラスタへのトラフィックに適用できます。

このタスクでは、`Gateway` を使ってサービスをサービスメッシュ外部に公開する方法を説明します。

{{< boilerplate gateway-api-support >}}

## 始める前に {#before-you-begin}

- [インストールガイド](/ja/docs/setup/) の手順に従って Istio をインストールしてください。

  {{< tip >}}
  `Gateway API` を使う場合は `minimal` プロファイルで Istio をインストールできます。
  この場合、`istio-ingressgateway` はデフォルトでインストールされません：

  {{< text bash >}}
  $ istioctl install --set profile=minimal
  {{< /text >}}

  {{< /tip >}}

- [httpbin]({{< github_tree >}}/samples/httpbin) サンプルを起動し、Ingress トラフィックのターゲットサービスとします：

  {{< text bash >}}
    $ kubectl apply -f @samples/httpbin/httpbin.yaml@
  {{< /text >}}

  ここでは「Kubernetes クラスタ」への Ingress トラフィック制御の例を示します。
  Sidecar インジェクションの有無に関わらず `httpbin` サービスを起動できます（ターゲットサービスは Istio メッシュ内外どちらでも可）。

## Gateway を使った Ingress の構成 {#configuring-ingress-using-a-gateway}

Ingress の `Gateway` はメッシュ境界で動作するロードバランサで、受信する HTTP/TCP 接続のポートやプロトコルなどを定義します。
[Kubernetes Ingress リソース](https://kubernetes.io/ja/docs/concepts/services-networking/ingress/) とは異なり、トラフィックルーティングは含まず、ルーティングは別途ルールで構成します（内部サービスリクエストと同様）。

ここでは 80 番ポートの HTTP トラフィック用に `Gateway` を構成する例を示します。

{{< tabset category-name="config-api" >}}

{{< tab name="Istio API" category-value="istio-apis" >}}

[Istio Gateway](/ja/docs/reference/config/networking/gateway/) を作成：

{{< text bash >}}
$ kubectl apply -f - <<EOF
apiVersion: networking.istio.io/v1
kind: Gateway
metadata:
name: httpbin-gateway
spec:

# selector は Ingress Gateway Pod のラベルと一致させる必要があります。

# 標準ドキュメント通り Helm でインストールした場合は "istio=ingress" に設定します。

selector:
istio: ingressgateway
servers:

- port:
  number: 80
  name: http
  protocol: HTTP
  hosts: - "httpbin.example.com"
  EOF
  {{< /text >}}

`Gateway` で受信トラフィックのルーティングを構成：

{{< text bash >}}
$ kubectl apply -f - <<EOF
apiVersion: networking.istio.io/v1
kind: VirtualService
metadata:
name: httpbin
spec:
hosts:

- "httpbin.example.com"
  gateways:
- httpbin-gateway
  http:
- match: - uri:
  prefix: /status - uri:
  prefix: /delay
  route: - destination:
  port:
  number: 8000
  host: httpbin
  EOF
  {{< /text >}}

`httpbin` サービス用の[VirtualService](/ja/docs/reference/config/networking/virtual-service/) を作成し、2 つのルールで `/status` と `/delay` パスへのトラフィックを許可します。

[Gateway](/ja/docs/reference/config/networking/virtual-service/#VirtualService-gateways) リストで `httpbin-gateway` 経由のリクエストのみ許可し、それ以外の外部リクエストは 404 で拒否されます。

{{< warning >}}
メッシュ内の他サービスからの内部リクエストはこれらのルールを通らず、デフォルトでラウンドロビンルールが適用されます。
`gateways` リストに `mesh` を追加すれば内部呼び出しにも同じルールを適用できます。
内部ホスト名（例：`httpbin.default.svc.cluster.local`）も `hosts` に追加してください。
詳細は[運用ガイド](/ja/docs/ops/common-problems/network-issues#route-rules-have-no-effect-on-ingress-gateway-requests)を参照してください。
{{< /warning >}}

{{< /tab >}}

{{< tab name="Gateway API" category-value="gateway-api" >}}

[Kubernetes Gateway](https://gateway-api.sigs.k8s.io/references/spec/#gateway.networking.k8s.io/v1.Gateway) を作成：

{{< text bash >}}
$ kubectl apply -f - <<EOF
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
name: httpbin-gateway
spec:
gatewayClassName: istio
listeners:

- name: http
  hostname: "httpbin.example.com"
  port: 80
  protocol: HTTP
  allowedRoutes:
  namespaces:
  from: Same
  EOF
  {{< /text >}}

{{< tip >}}
本番環境では `Gateway` とルートは異なるロールのユーザーが別々の名前空間で作成することが多いです。
この場合、`Gateway` の `allowedRoutes` でルート作成を許可する名前空間を指定します。
この例ではルートも `Gateway` と同じ名前空間に作成しています。
{{< /tip >}}

Kubernetes `Gateway` リソースの作成は[関連プロキシサービスのデプロイ](/ja/docs/tasks/traffic-management/ingress/gateway-api/#automated-deployment)も伴うため、次のコマンドで Gateway の準備完了を待ちます：

{{< text bash >}}
$ kubectl wait --for=condition=programmed gtw httpbin-gateway
{{< /text >}}

`Gateway` で受信トラフィックのルーティングを構成：

{{< text bash >}}
$ kubectl apply -f - <<EOF
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
name: httpbin
spec:
parentRefs:

- name: httpbin-gateway
  hostnames: ["httpbin.example.com"]
  rules:
- matches: - path:
  type: PathPrefix
  value: /status - path:
  type: PathPrefix
  value: /delay
  backendRefs: - name: httpbin
  port: 8000
  EOF
  {{< /text >}}

`httpbin` サービス用の[HTTPRoute](https://gateway-api.sigs.k8s.io/references/spec/#gateway.networking.k8s.io/v1.HTTPRoute) を作成し、2 つのルールで `/status` と `/delay` パスへのトラフィックを許可します。

{{< /tab >}}

{{< /tabset >}}

### Ingress IP とポートの確認 {#determining-the-ingress-ip-and-ports}

各 `Gateway` は [LoadBalancer タイプの Service](https://kubernetes.io/ja/docs/tasks/access-application-cluster/create-external-load-balancer/) によって支えられ、この Service の外部負荷分散器の IP とポートが Gateway にアクセスするために使用されます。
ほとんどのクラウドプラットフォームで実行されるクラスターはデフォルトで `LoadBalancer` タイプの Kubernetes Service をサポートしますが、一部の環境（例：テスト環境）では以下の操作が必要になる場合があります：

- `minikube` - 別のターミナルで以下のコマンドを実行して外部負荷分散器を起動します：

  {{< text syntax=bash snip_id=minikube_tunnel >}}
    $ minikube tunnel
  {{< /text >}}

- `kind` - `LoadBalancer` タイプの Service を[ガイド](https://kind.sigs.k8s.io/docs/user/loadbalancer/)に従って正常に動作させます。

- その他のプラットフォーム - [MetalLB](https://metallb.universe.tf/installation/) を使用して `LoadBalancer` Service の `EXTERNAL-IP` を取得できます。

デモンストレーションを容易にするために、Ingress IP とポートを環境変数に格納し、後続のチュートリアルで使用します。以下の指示に従って `INGRESS_HOST` と `INGRESS_PORT` 環境変数を設定します：

{{< tabset category-name="config-api" >}}

{{< tab name="Istio API" category-value="istio-apis" >}}

以下の環境変数をクラスター内の Istio Ingress Gateway の名前とその名前空間に設定します：

{{< text bash >}}
$ export INGRESS_NAME=istio-ingressgateway
$ export INGRESS_NS=istio-system
{{< /text >}}

{{< tip >}}
Helm で Istio をインストールした場合、Ingress Gateway の名前と名前空間は `istio-ingress` です：

{{< text bash >}}
$ export INGRESS_NAME=istio-ingress
$ export INGRESS_NS=istio-ingress
{{< /text >}}

{{< /tip >}}

以下のコマンドを実行して、Kubernetes クラスターが外部負荷分散器をサポートしているかどうかを確認します：

{{< text bash >}}
$ kubectl get svc "$INGRESS_NAME" -n "$INGRESS_NS"
NAME TYPE CLUSTER-IP EXTERNAL-IP PORT(S) AGE
istio-ingressgateway LoadBalancer 172.21.109.129 130.211.10.121 ... 17h
{{< /text >}}

`EXTERNAL-IP` の値が設定されている場合、その環境は Ingress Gateway に外部負荷分散器を提供しています。`EXTERNAL-IP` の値が `<none>`（または `<pending>` のまま）の場合、その環境は Ingress Gateway に外部負荷分散器を提供していません。

外部負荷分散器をサポートしていない環境の場合は、[Ingress Gateway サービスの Node Port を使用して Ingress Gateway にアクセス](/ja/docs/tasks/traffic-management/ingress/ingress-control/#using-node-ports-of-the-ingress-gateway-service)を試みてください。そうでない場合は、以下のコマンドで Ingress IP とポートを設定します：

{{< text bash >}}
$ export INGRESS_HOST=$(kubectl -n "$INGRESS_NS" get service "$INGRESS_NAME" -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
$ export INGRESS_PORT=$(kubectl -n "$INGRESS_NS" get service "$INGRESS_NAME" -o jsonpath='{.spec.ports[?(@.name=="http2")].port}')
$ export SECURE_INGRESS_PORT=$(kubectl -n "$INGRESS_NS" get service "$INGRESS_NAME" -o jsonpath='{.spec.ports[?(@.name=="https")].port}')
$ export TCP_INGRESS_PORT=$(kubectl -n "$INGRESS_NS" get service "$INGRESS_NAME" -o jsonpath='{.spec.ports[?(@.name=="tcp")].port}')
{{< /text >}}

{{< warning >}}
特定の環境では、ホスト名ではなく IP アドレスを使用して負荷分散器を公開する場合があります。この場合、Ingress Gateway の `EXTERNAL-IP` は IP アドレスではなくホスト名になります。上記の `INGRESS_HOST` 環境変数の設定コマンドは失敗します。`INGRESS_HOST` の値を以下のように修正できます：

{{< text bash >}}
$ export INGRESS_HOST=$(kubectl -n "$INGRESS_NS" get service "$INGRESS_NAME" -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')
{{< /text >}}

{{< /warning >}}

{{< /tab >}}

{{< tab name="Gateway API" category-value="gateway-api" >}}

httpbin ゲートウェイリソースからゲートウェイアドレスとポートを取得します：

{{< text bash >}}
$ export INGRESS_HOST=$(kubectl get gtw httpbin-gateway -o jsonpath='{.status.addresses[0].value}')
$ export INGRESS_PORT=$(kubectl get gtw httpbin-gateway -o jsonpath='{.spec.listeners[?(@.name=="http")].port}')
{{< /text >}}

{{< tip >}}
同様のコマンドで他のゲートウェイの他のポートにアクセスできます。例えば、名前が `my-gateway` のゲートウェイで名前が `https` の安全な HTTP ポートにアクセスする場合：

{{< text bash >}}
$ export INGRESS_HOST=$(kubectl get gtw my-gateway -o jsonpath='{.status.addresses[0].value}')
$ export SECURE_INGRESS_PORT=$(kubectl get gtw my-gateway -o jsonpath='{.spec.listeners[?(@.name=="https")].port}')
{{< /text >}}

{{< /tip >}}

{{< /tab >}}

{{< /tabset >}}

## Ingress サービスへのアクセス {#accessing-ingress-services}

1. **curl** を使用して **httpbin** サービスにアクセスします：

   {{< text bash >}}
    $ curl -s -I -HHost:httpbin.example.com "http://$INGRESS_HOST:$INGRESS_PORT/status/200"
    ...
    HTTP/1.1 200 OK
    ...
    server: istio-envoy
    ...
   {{< /text >}}

   このコマンドは `-H` フラグを使用して HTTP ヘッダーパラメータ **Host** を "httpbin.example.com" に設定します。
   この操作は必須です。Ingress `Gateway` は "httpbin.example.com" サービスリクエストを処理するように構成されているため、テスト環境ではホスト名に DNS をバインドせず、単純に Ingress IP にリクエストを送信するためです。

1. 明示的に公開されていない URL にアクセスすると、HTTP 404 エラーが表示されます：

   {{< text bash >}}
    $ curl -s -I -HHost:httpbin.example.com "http://$INGRESS_HOST:$INGRESS_PORT/headers"
    HTTP/1.1 404 Not Found
    ...
   {{< /text >}}

## Ingress サービスへのブラウザアクセス {#accessing-ingress-services-using-a-browser}

ブラウザで `httpbin` サービスの URL にアクセスしても有効な応答が得られません。`curl` のように、ブラウザはリクエストヘッダーパラメータ **Host** を渡すことができないためです。実際のシナリオでは、リクエストを処理するホストと解決可能な DNS を適切に構成する必要があり、URL でホストのドメインを使用します（例：`https://httpbin.example.com/status/200`）。

以下のように、この問題を回避するために、簡単なテストとデモンストレーションでは以下の方法を使用できます：

{{< tabset category-name="config-api" >}}

{{< tab name="Istio API" category-value="istio-apis" >}}

`Gateway` と `VirtualService` の構成で `*` を使用します。例えば、以下のように Ingress 構成を変更します：

{{< text bash >}}
$ kubectl apply -f - <<EOF
apiVersion: networking.istio.io/v1
kind: Gateway
metadata:
name: httpbin-gateway
spec:

# selector は Ingress Gateway Pod のラベルと一致させる必要があります。

# 標準ドキュメント通り Helm でインストールした場合は "istio=ingress" に設定します。

selector:
istio: ingressgateway
servers:

- port:
  number: 80
  name: http
  protocol: HTTP
  hosts:
  - "\*"

---

apiVersion: networking.istio.io/v1
kind: VirtualService
metadata:
name: httpbin
spec:
hosts:

- "\*"
  gateways:
- httpbin-gateway
  http:
- match: - uri:
  prefix: /headers
  route: - destination:
  port:
  number: 8000
  host: httpbin
  EOF
  {{< /text >}}

{{< /tab >}}

{{< tab name="Gateway API" category-value="gateway-api" >}}

`Gateway` と `HTTPRoute` の構成からホスト名を削除すると、この操作がすべてのリクエストに適用されます。例えば、以下のように Ingress 構成を変更します：

{{< text bash >}}
$ kubectl apply -f - <<EOF
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
name: httpbin-gateway
spec:
gatewayClassName: istio
listeners:

- name: http
  port: 80
  protocol: HTTP
  allowedRoutes:
  namespaces:
  from: Same

---

apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
name: httpbin
spec:
parentRefs:

- name: httpbin-gateway
  rules:
- matches: - path:
  type: PathPrefix
  value: /headers
  backendRefs: - name: httpbin
  port: 8000
  EOF
  {{< /text >}}

{{< /tab >}}

{{< /tabset >}}

この時点で、ブラウザで `$INGRESS_HOST:$INGRESS_PORT` を含む URL を入力できます。例えば、`http://$INGRESS_HOST:$INGRESS_PORT/headers` と入力すると、ブラウザが送信したすべてのヘッダー情報が表示されます。

## 原理の理解 {#understanding-what-happened}

`Gateway` 構成リソースは外部トラフィックを Istio サービスメッシュに入れ、境界サービスに対してトラフィック管理と Istio で利用可能なポリシー特性を適用します。

前のステップでは、サービスメッシュ内にサービスを作成し、外部トラフィックに対して HTTP エンドポイントを公開しました。

## Ingress Gateway サービスの Node Port の使用 {#using-node-ports-of-the-ingress-gateway-service}

{{< warning >}}
Kubernetes 環境に[LoadBalancer タイプの Service](https://kubernetes.io/ja/docs/tasks/access-application-cluster/create-external-load-balancer/) がサポートされている場合は、これらの指示手順を省略できます。
{{< /warning >}}

外部負荷分散器をサポートしていない環境では、`istio-ingressgateway` Service の [Node Port](https://kubernetes.io/ja/docs/concepts/services-networking/service/#type-nodeport) を使用して Istio の特定の機能を実験できます。

Ingress ポートを設定します：

{{< text bash >}}
$ export INGRESS_PORT=$(kubectl -n "${INGRESS_NS}" get service "${INGRESS_NAME}" -o jsonpath='{.spec.ports[?(@.name=="http2")].nodePort}')
$ export SECURE_INGRESS_PORT=$(kubectl -n "${INGRESS_NS}" get service "${INGRESS_NAME}" -o jsonpath='{.spec.ports[?(@.name=="https")].nodePort}')
$ export TCP_INGRESS_PORT=$(kubectl -n "${INGRESS_NS}" get service "${INGRESS_NAME}" -o jsonpath='{.spec.ports[?(@.name=="tcp")].nodePort}')
{{< /text >}}

クラスター提供者の要件に従って Ingress IP を設定します：

1.  **GKE：**

    {{< text bash >}}
    $ export INGRESS_HOST=worker-node-address
    {{< /text >}}

    ファイアウォールルールを作成して、**ingressgateway** Service のポートへの TCP トラフィックを許可する必要があります。HTTP および/または HTTPS ポートへのトラフィックを許可するコマンドを実行します：

    {{< text bash >}}
    $ gcloud compute firewall-rules create allow-gateway-http --allow "tcp:$INGRESS_PORT"
    $ gcloud compute firewall-rules create allow-gateway-https --allow "tcp:$SECURE_INGRESS_PORT"
    {{< /text >}}

1.  **IBM Cloud Kubernetes Service：**

    {{< text bash >}}
    $ ibmcloud ks workers --cluster cluster-name-or-id
    $ export INGRESS_HOST=public-IP-of-one-of-the-worker-nodes
    {{< /text >}}

1.  **Docker For Desktop：**

    {{< text bash >}}
    $ export INGRESS_HOST=127.0.0.1
    {{< /text >}}

1.  **その他の環境：**

    {{< text bash >}}
    $ export INGRESS_HOST=$(kubectl get po -l istio=ingressgateway -n "${INGRESS_NS}" -o jsonpath='{.items[0].status.hostIP}')
    {{< /text >}}

## トラブルシューティング {#troubleshooting}

1. 環境変数 `INGRESS_HOST` と `INGRESS_PORT` の値を確認してください。これらが正当な値であることを確認するには、以下のコマンドを実行します：

   {{< text bash >}}
    $ kubectl get svc -n istio-system
    $ echo "INGRESS_HOST=$INGRESS_HOST, INGRESS_PORT=$INGRESS_PORT"
   {{< /text >}}

1. 同じポートで別の Istio Ingress Gateway が定義されていないか確認してください：

   {{< text bash >}}
    $ kubectl get gateway --all-namespaces
   {{< /text >}}

1. 同じ IP とポートで Kubernetes Ingress リソースが定義されていないか確認してください：

   {{< text bash >}}
    $ kubectl get ingress --all-namespaces
   {{< /text >}}

1. 外部負荷分散器を使用したが正常に動作しない場合は、[その Node Port を使用して Gateway にアクセス](/ja/docs/tasks/traffic-management/ingress/ingress-control/#using-node-ports-of-the-ingress-gateway-service)を試みてください。

## クリーンアップ {#cleanup}

{{< tabset category-name="config-api" >}}

{{< tab name="Istio API" category-value="istio-apis" >}}

`Gateway` と `VirtualService` の構成を削除し、
[httpbin]({{< github_tree >}}/samples/httpbin) サービスを終了します：

{{< text bash >}}
$ kubectl delete gateway httpbin-gateway
$ kubectl delete virtualservice httpbin
$ kubectl delete --ignore-not-found=true -f @samples/httpbin/httpbin.yaml@
{{< /text >}}

{{< /tab >}}

{{< tab name="Gateway API" category-value="gateway-api" >}}

`Gateway` と `HTTPRoute` の構成を削除し、
[httpbin]({{< github_tree >}}/samples/httpbin) サービスを終了します：

{{< text bash >}}
$ kubectl delete httproute httpbin
$ kubectl delete gtw httpbin-gateway
$ kubectl delete --ignore-not-found=true -f @samples/httpbin/httpbin.yaml@
{{< /text >}}

{{< /tab >}}

{{< /tabset >}}
