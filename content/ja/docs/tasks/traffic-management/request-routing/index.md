---
title: リクエストルーティングの設定
description: リクエストをマイクロサービスの複数バージョンへ動的にルーティングする方法。
weight: 10
aliases:
  - /zh/docs/tasks/request-routing.html
keywords: [traffic-management, routing]
owner: istio/wg-networking-maintainers
test: yes
---

このタスクでは、リクエストをマイクロサービスの複数バージョンへ動的にルーティングする方法を紹介します。

{{< boilerplate gateway-api-support >}}

## 始める前に {#before-you-begin}

- [インストールガイド](/ja/docs/setup/) の手順に従って Istio をインストールします。

- [Bookinfo](/ja/docs/examples/bookinfo/) サンプルアプリケーションをデプロイします。

- [トラフィック管理](/ja/docs/concepts/traffic-management) の概念ドキュメントを確認してください。このタスクを試す前に、**Destination Rule**、**Virtual Service**、**Subset** などの重要な用語に慣れておく必要があります。

## このタスクについて {#about-this-task}

Istio の [Bookinfo](/ja/docs/examples/bookinfo/) サンプルには 4 つの独立したマイクロサービスが含まれており、それぞれに複数のバージョンがあります。そのうちの 1 つ、`reviews` マイクロサービスには 3 つの異なるバージョンがデプロイされ、同時に稼働しています。
この問題を説明するために、ブラウザで Bookinfo アプリケーションの `/productpage` にアクセスし、何度かリロードしてみてください。
URL は `http://$GATEWAY_URL/productpage` です。`$GATEWAY_URL` は Ingress の外部アクセス IP アドレスで、[Bookinfo](/ja/docs/examples/bookinfo/#determine-the-ingress-ip-and-port) ドキュメントで説明されています。

書評の出力に星評価が表示されたりされなかったりすることに気づくでしょう。これは、デフォルトのサービスバージョンが明示的にルーティングされていないため、Istio が利用可能なすべてのバージョンにリクエストをラウンドロビンでルーティングしているためです。

このタスクの最初の目標は、すべてのトラフィックをマイクロサービスの `v1`（バージョン 1）にルーティングするルールを適用することです。後ほど、HTTP リクエストヘッダーの値に基づいてトラフィックをルーティングするルールを適用します。

## バージョン 1 へのルーティング {#route-to-version-1}

1 つのバージョンのみにルーティングするには、Virtual Service を使ってマイクロサービスのデフォルトバージョンを設定します。

{{< warning >}}
サービスバージョンがまだ定義されていない場合は、[サービスバージョンの定義](/ja/docs/examples/bookinfo/#define-the-service-versions) の手順に従ってください。
{{< /warning >}}

1. 次のコマンドを実行してルーティングルールを作成します：
   {{< tabset category-name="config-api" >}}

{{< tab name="Istio APIs" category-value="istio-apis" >}}

Istio では Virtual Service を使ってルーティングルールを定義します。
次のコマンドで Virtual Service を適用します。
この場合、Virtual Service はすべてのトラフィックを各マイクロサービスの `v1` バージョンにルーティングします。

{{< text bash >}}
$ kubectl apply -f @samples/bookinfo/networking/virtual-service-all-v1.yaml@
{{< /text >}}

構成の伝播は最終的に一貫性があるため、Virtual Service が有効になるまで数秒待ってください。

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
- backendRefs: - name: reviews-v1
  port: 9080
  EOF
  {{< /text >}}

{{< /tab >}}

{{< /tabset >}}

2. 次のコマンドで定義済みのルーティングを表示します：

{{< tabset category-name="config-api" >}}

{{< tab name="Istio APIs" category-value="istio-apis" >}}

{{< text bash yaml >}}
$ kubectl get virtualservices -o yaml

- apiVersion: networking.istio.io/v1
  kind: VirtualService
  ...
  spec:
  hosts:
  - details
    http:
  - route:
    - destination:
      host: details
      subset: v1
- apiVersion: networking.istio.io/v1
  kind: VirtualService
  ...
  spec:
  hosts:
  - productpage
    http:
  - route:
    - destination:
      host: productpage
      subset: v1
- apiVersion: networking.istio.io/v1
  kind: VirtualService
  ...
  spec:
  hosts:
  - ratings
    http:
  - route:
    - destination:
      host: ratings
      subset: v1
- apiVersion: networking.istio.io/v1
  kind: VirtualService
  ...
  spec:
  hosts: - reviews
  http: - route: - destination:
  host: reviews
  subset: v1
  {{< /text >}}

また、次のコマンドで対応する `subset` 定義を表示できます：

{{< text bash >}}
$ kubectl get destinationrules -o yaml
{{< /text >}}

{{< /tab >}}

{{< tab name="Gateway API" category-value="gateway-api" >}}

{{< text bash >}}
$ kubectl get httproute reviews -o yaml
...
spec:
parentRefs:

- group: gateway.networking.k8s.io
  kind: Service
  name: reviews
  port: 9080
  rules:
- backendRefs: - group: ""
  kind: Service
  name: reviews-v1
  port: 9080
  weight: 1
  matches: - path:
  type: PathPrefix
  value: /
  status:
  parents:
- conditions: - lastTransitionTime: "2022-11-08T19:56:19Z"
  message: Route was valid
  observedGeneration: 8
  reason: Accepted
  status: "True"
  type: Accepted - lastTransitionTime: "2022-11-08T19:56:19Z"
  message: All references resolved
  observedGeneration: 8
  reason: ResolvedRefs
  status: "True"
  type: ResolvedRefs
  controllerName: istio.io/gateway-controller
  parentRef:
  group: gateway.networking.k8s.io
  kind: Service
  name: reviews
  port: 9080
  {{< /text >}}

リソースの状態で、`reviews` 親の `Accepted` 条件が `True` であることを確認してください。

{{< /tab >}}

{{< /tabset >}}

Istio を Bookinfo マイクロサービスの `v1` バージョン、特に `reviews` サービスのバージョン 1 へルーティングするように構成しました。

## 新しいルーティング構成のテスト {#test-the-new-routing-configuration}

Bookinfo アプリケーションの `/productpage` を再度リロードすることで、新しい構成を簡単にテストできます。
ブラウザで Bookinfo サイトを開きます。URL は `http://$GATEWAY_URL/productpage` で、`$GATEWAY_URL` は外部の Ingress IP アドレスです（[Bookinfo](/ja/docs/examples/bookinfo/#determine-the-ingress-IP-and-port) ドキュメント参照）。何度リロードしても、ページのレビュー部分に星評価が表示されないことに注意してください。これは、Istio を `reviews:v1` バージョンにすべてのトラフィックをルーティングするように構成したためで、このバージョンは星評価サービスへアクセスしません。

このタスクの第 1 部：トラフィックを 1 つのサービスバージョンへルーティングする、が完了しました。

## ユーザー ID に基づくルーティング {#route-based-on-user-identity}

次に、特定のユーザーからのすべてのトラフィックを特定のサービスバージョンへルーティングするようにルーティング構成を変更します。この例では、ユーザー名が Jason のすべてのトラフィックを `reviews:v2` サービスへルーティングします。

Istio にはユーザー ID に特化した仕組みはありません。実際には、`productpage` サービスが `reviews` サービスへのすべての HTTP リクエストにカスタム `end-user` リクエストヘッダーを追加することで、この例の動作を実現しています。

Istio はまた、エントリーゲートウェイで強力な認証 JWT に基づくルーティングもサポートしています。詳細は [JWT クレームベースルーティング](/ja/docs/tasks/security/authentication/jwt-route) を参照してください。

`reviews:v2` は星評価機能を含むバージョンです。

1. 次のコマンドを実行してユーザーベースのルーティングを有効にします：

{{< tabset category-name="config-api" >}}

{{< tab name="Istio APIs" category-value="istio-apis" >}}

{{< text bash >}}
$ kubectl apply -f @samples/bookinfo/networking/virtual-service-reviews-test-v2.yaml@
{{< /text >}}

次のコマンドでルールが作成されたことを確認できます：

{{< text bash yaml >}}
$ kubectl get virtualservice reviews -o yaml
apiVersion: networking.istio.io/v1
kind: VirtualService
...
spec:
hosts:

- reviews
  http:
- match:
  - headers:
    end-user:
    exact: jason
    route:
  - destination:
    host: reviews
    subset: v2
- route: - destination:
  host: reviews
  subset: v1
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
- matches:
  - headers:
    - name: end-user
      value: jason
      backendRefs:
  - name: reviews-v2
    port: 9080
- backendRefs: - name: reviews-v1
  port: 9080
  EOF
  {{< /text >}}

{{< /tab >}}

{{< /tabset >}}

2. Bookinfo アプリケーションの `/productpage` でユーザー `jason` としてログインします。

   ブラウザをリロードしてください。何が見えますか？各レビューの横に星評価が表示されます。

3. 他のユーザー名でログインします（任意の名前を選択してください）。

   ブラウザをリロードしてください。今度は星が消えます。これは Jason 以外のすべてのユーザーのトラフィックが `reviews:v1` にルーティングされるためです。

Istio でユーザー ID に基づくトラフィックルーティングが正しく構成されました。

## 仕組みの理解 {#understanding-what-happened}

このタスクでは、まず Istio を使って Bookinfo サービスの `v1` バージョンに 100% のリクエストトラフィックをルーティングしました。
次に、`productpage` サービスからのリクエストに含まれる `end-user` カスタムリクエストヘッダーの内容に基づいて、特定のトラフィックを `reviews` サービスの `v2` バージョンに選択的にルーティングするルールを設定しました。

Kubernetes のサービス（このタスクで使われている Bookinfo サービスなど）は、Istio の L7 ルーティング機能を活用するために特定の要件を満たす必要があります。詳細は [Pod と Service の要件](/ja/docs/ops/deployment/application-requirements/) を参照してください。

[トラフィックシフト](/ja/docs/tasks/traffic-management/traffic-shifting) タスクでは、ここで学んだのと同じ基本パターンでルーティングルールを設定し、サービスのあるバージョンから別のバージョンへ段階的にトラフィックを移行します。

## クリーンアップ {#cleanup}

1. アプリケーションのルーティングルールを削除します：

{{< tabset category-name="config-api" >}}

{{< tab name="Istio APIs" category-value="istio-apis" >}}

{{< text bash >}}
$ kubectl delete -f @samples/bookinfo/networking/virtual-service-all-v1.yaml@
{{< /text >}}

{{< /tab >}}

{{< tab name="Gateway API" category-value="gateway-api" >}}

{{< text bash >}}
$ kubectl delete httproute reviews
{{< /text >}}

{{< /tab >}}

{{< /tabset >}}

2. 今後のタスクを試す予定がない場合は、[Bookinfo のクリーンアップ](/ja/docs/examples/bookinfo/#cleanup) の手順に従ってアプリケーションを停止してください。
