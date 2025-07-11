---
title: トラフィックシフト
description: トラフィックを旧バージョンから新バージョンのサービスへ移行する方法を紹介します。
weight: 30
keywords: [traffic-management, traffic-shifting]
aliases:
  - /zh/docs/tasks/traffic-management/version-migration.html
owner: istio/wg-networking-maintainers
test: yes
---

このタスクでは、マイクロサービスのあるバージョンから別のバージョンへトラフィックを段階的に移行する方法を紹介します。
たとえば、トラフィックを旧バージョンから新バージョンへ移行できます。

よくあるユースケースは、マイクロサービスのあるバージョンから別のバージョンへトラフィックを段階的に移行することです。Istio では、一連のルールを設定することで、トラフィックの一定割合を一方または他方のサービスへルーティングできます。

このタスクでは、まずトラフィックの 50％ を `reviews:v1` に、残りの 50％ を `reviews:v3` に送信します。その後、100％ のトラフィックを `reviews:v3` に送信して移行を完了します。

{{< boilerplate gateway-api-support >}}

## 始める前に {#before-you-begin}

- [インストールガイド](/ja/docs/setup/) の手順に従って Istio をインストールします。

- [Bookinfo](/ja/docs/examples/bookinfo/) サンプルアプリケーションをデプロイします。

- [トラフィック管理](/ja/docs/concepts/traffic-management) の概念ドキュメントを確認してください。

## 重み付けルーティングの適用 {#apply-weight-based-routing}

{{< warning >}}
サービスバージョンがまだ定義されていない場合は、[サービスバージョンの定義](/ja/docs/examples/bookinfo/#define-the-service-versions) の手順に従ってください。
{{< /warning >}}

1. まず、次のコマンドで全トラフィックを各マイクロサービスの `v1` バージョンにルーティングします。

{{< tabset category-name="config-api" >}}

{{< tab name="Istio APIs" category-value="istio-apis" >}}

{{< text syntax=bash snip_id=config_all_v1 >}}
$ kubectl apply -f @samples/bookinfo/networking/virtual-service-all-v1.yaml@
{{< /text >}}

{{< /tab >}}

{{< tab name="Gateway API" category-value="gateway-api" >}}

{{< text syntax=bash snip_id=gtw_config_all_v1 >}}
$ kubectl apply -f @samples/bookinfo/gateway-api/route-reviews-v1.yaml@
{{< /text >}}

{{< /tab >}}

{{< /tabset >}}

2. ブラウザで Bookinfo サイトを開きます。URL は `http://$GATEWAY_URL/productpage` です。
   `$GATEWAY_URL` は Ingress の外部 IP アドレスで、[Bookinfo](/ja/docs/examples/bookinfo/#determine-the-ingress-IP-and-port) ドキュメントで説明されています。

   何度リロードしても、ページのレビュー部分に星評価が表示されないことに注意してください。
   これは、Istio が星評価サービスの全トラフィックを `reviews:v1` バージョンにルーティングするよう構成されており、このバージョンは星評価サービスへアクセスしないためです。

3. 次のコマンドで `reviews:v1` から `reviews:v3` へ 50% のトラフィックを移行します：

{{< tabset category-name="config-api" >}}

{{< tab name="Istio APIs" category-value="istio-apis" >}}

{{< text syntax=bash snip_id=config_50_v3 >}}
$ kubectl apply -f @samples/bookinfo/networking/virtual-service-reviews-50-v3.yaml@
{{< /text >}}

{{< /tab >}}

{{< tab name="Gateway API" category-value="gateway-api" >}}

{{< text syntax=bash snip_id=gtw_config_50_v3 >}}
$ kubectl apply -f @samples/bookinfo/gateway-api/route-reviews-50-v3.yaml@
{{< /text >}}

{{< /tab >}}

{{< /tabset >}}

4. 数秒待って新しいルールが伝播され、ルールが置き換わったことを確認します：

{{< tabset category-name="config-api" >}}

{{< tab name="Istio APIs" category-value="istio-apis" >}}

{{< text syntax=bash outputis=yaml snip_id=verify_config_50_v3 >}}
$ kubectl get virtualservice reviews -o yaml
apiVersion: networking.istio.io/v1
kind: VirtualService
...
spec:
hosts:

- reviews
  http:
- route: - destination:
  host: reviews
  subset: v1
  weight: 50 - destination:
  host: reviews
  subset: v3
  weight: 50
  {{< /text >}}

{{< /tab >}}

{{< tab name="Gateway API" category-value="gateway-api" >}}

{{< text syntax=bash outputis=yaml snip_id=gtw_verify_config_50_v3 >}}
$ kubectl get httproute reviews -o yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
...
spec:
parentRefs:

- group: ""
  kind: Service
  name: reviews
  port: 9080
  rules:
- backendRefs: - group: ""
  kind: Service
  name: reviews-v1
  port: 9080
  weight: 50 - group: ""
  kind: Service
  name: reviews-v3
  port: 9080
  weight: 50
  matches: - path:
  type: PathPrefix
  value: /
  status:
  parents:
- conditions: - lastTransitionTime: "2022-11-10T18:13:43Z"
  message: Route was valid
  observedGeneration: 14
  reason: Accepted
  status: "True"
  type: Accepted
  ...
  {{< /text >}}

{{< /tab >}}

{{< /tabset >}}

5. ブラウザで `/productpage` ページをリロードすると、約 50% の確率で**赤い**星評価が表示されます。
   これは `reviews` の `v3` バージョンが星評価を表示でき、`v1` バージョンはできないためです。

   {{< tip >}}
   現在の Envoy Sidecar 実装では、正しいトラフィック分配の効果を見るには `/productpage` を何度も（15 回以上）リロードする必要があるかもしれません。ルールを変更して 90% のトラフィックを `v3` にルーティングすると、より多くの赤い星評価が見られます。
   {{< /tip >}}

6. `reviews:v3` マイクロサービスが安定したと判断したら、Virtual Service ルールを適用して 100% のトラフィックを `reviews:v3` にルーティングできます：

{{< tabset category-name="config-api" >}}

{{< tab name="Istio APIs" category-value="istio-apis" >}}

{{< text syntax=bash snip_id=config_100_v3 >}}
$ kubectl apply -f @samples/bookinfo/networking/virtual-service-reviews-v3.yaml@
{{< /text >}}

{{< /tab >}}

{{< tab name="Gateway API" category-value="gateway-api" >}}

{{< text syntax=bash snip_id=gtw_config_100_v3 >}}
$ kubectl apply -f @samples/bookinfo/gateway-api/route-reviews-v3.yaml@
{{< /text >}}

{{< /tab >}}

{{< /tabset >}}

7. これで `/productpage` をリロードすると、常に**赤い**星評価付きのレビューが表示されます。

## 仕組みの理解 {#understanding-what-happened}

このタスクでは、Istio の重み付けルーティング機能を使って `reviews` サービスのトラフィックを新バージョンへ移行しました。これは、コンテナオーケストレーションプラットフォームのデプロイ機能によるバージョン移行とは異なります。後者はインスタンスのスケールアウトによってトラフィックを管理します。

Istio では、2 つの `reviews` サービスバージョンを個別にスケールイン・スケールアウトでき、この操作は両バージョン間のトラフィック分配に影響しません。

自動スケーリング対応のバージョンルーティングの詳細は、[Istio でのカナリアリリース](/ja/blog/2017/0.1-canary/) をご覧ください。

## クリーンアップ {#cleanup}

1. アプリケーションのルーティングルールを削除します。

{{< tabset category-name="config-api" >}}

{{< tab name="Istio APIs" category-value="istio-apis" >}}

{{< text syntax=bash snip_id=cleanup >}}
$ kubectl delete -f @samples/bookinfo/networking/virtual-service-all-v1.yaml@
{{< /text >}}

{{< /tab >}}

{{< tab name="Gateway API" category-value="gateway-api" >}}

{{< text syntax=bash snip_id=gtw_cleanup >}}
$ kubectl delete httproute reviews
{{< /text >}}

{{< /tab >}}

{{< /tabset >}}

2. 今後のタスクを試す予定がない場合は、[Bookinfo のクリーンアップ](/ja/docs/examples/bookinfo/#cleanup) の手順に従ってアプリケーションを停止してください。
