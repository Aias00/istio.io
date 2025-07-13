---
title: Kubernetes Gateway API
description: Istio で Kubernetes Gateway API を構成する方法を説明します。
weight: 50
aliases:
  - /zh/docs/tasks/traffic-management/ingress/service-apis/
  - /latest/zh/docs/tasks/traffic-management/ingress/service-apis/
keywords: [traffic-management, ingress, gateway-api]
owner: istio/wg-networking-maintainers
test: yes
---

Istio 独自のトラフィック管理 API 以外にも、
{{< boilerplate gateway-api-future >}}
本記事では Istio と Kubernetes API の違いを説明し、Gateway API を使ってサービスメッシュクラスタ外部にサービスを公開する方法の簡単な例を紹介します。
これらの API は Kubernetes の [Service](https://kubernetes.io/ja/docs/concepts/services-networking/service/) や [Ingress](https://kubernetes.io/ja/docs/concepts/services-networking/ingress/) API の積極的な進化形です。

{{< tip >}}
多くの Istio トラフィック管理ドキュメントは Istio または Kubernetes API の使い方を説明しています（例：[インバウンドトラフィックの制御](/ja/docs/tasks/traffic-management/ingress/ingress-control) 参照）。
[はじめに](/ja/docs/setup/getting-started/) から Gateway API を使い始めることもできます。
{{< /tip >}}

## セットアップ {#setup}

1. 多くの Kubernetes クラスタでは Gateway API はデフォルトでインストールされていません。Gateway API CRD が存在しない場合は、次のコマンドでインストールします：

   {{< text bash >}}
    $ kubectl get crd gateways.gateway.networking.k8s.io &> /dev/null || \
     { kubectl kustomize "github.com/kubernetes-sigs/gateway-api/config/crd?ref={{< k8s_gateway_api_version >}}" | kubectl apply -f -; }
   {{< /text >}}

1. `minimal` プロファイルで Istio をインストールします：

   {{< text bash >}}
    $ istioctl install --set profile=minimal -y
   {{< /text >}}

## Istio API との違い {#differences-from-istio-apis}

Gateway API は Istio API（Gateway や VirtualService など）と多くの共通点があります。
主要リソースは同じ `Gateway` という名前を使い、似た目的で利用されます。

新しい Gateway API は、Kubernetes のさまざまな Ingress 実装（Istio を含む）からの知見をもとに、標準化されたベンダーニュートラルな API を構築することを目指しています。
これらの API は通常、Istio Gateway や VirtualService と同じ用途ですが、いくつか重要な違いがあります：

- Istio API の `Gateway` は[デプロイ済み](/ja/docs/setup/additional-setup/gateway/)の既存 Gateway Deployment/Service の**設定のみ**を行いますが、Gateway API の `Gateway` リソースは**設定とデプロイの両方**を行います。詳細は[デプロイ方法](#deployment-methods)を参照してください。
- Istio の `VirtualService` ではすべてのプロトコルを単一リソースで設定しますが、Gateway API ではプロトコルごとに個別リソース（`HTTPRoute` や `TCPRoute` など）を使います。
- Gateway API は豊富なルーティング機能を提供しますが、Istio の全機能をまだカバーしていません。今後 API の拡張や[拡張性](https://gateway-api.sigs.k8s.io/#gateway-api-concepts)を活かした Istio 機能の公開が進められています。

## ゲートウェイの構成 {#configuring-a-gateway}

API の詳細は [Gateway API](https://gateway-api.sigs.k8s.io/) ドキュメントを参照してください。

ここでは簡単なアプリをデプロイし、`Gateway` で外部公開する例を示します。

1. まず `httpbin` テストアプリをデプロイします：

   {{< text bash >}}
    $ kubectl apply -f @samples/httpbin/httpbin.yaml@
   {{< /text >}}

1. Gateway API 設定（`/get` のみ公開するルート）をデプロイします：

   {{< text bash >}}
    $ kubectl create namespace istio-ingress
    $ kubectl apply -f - <<EOF
    apiVersion: gateway.networking.k8s.io/v1
    kind: Gateway
    metadata:
    name: gateway
    namespace: istio-ingress
    spec:
    gatewayClassName: istio
    listeners:

    - name: default
      hostname: "\*.example.com"
      port: 80
      protocol: HTTP
      allowedRoutes:
      namespaces:
      from: All

    ***

    apiVersion: gateway.networking.k8s.io/v1
    kind: HTTPRoute
    metadata:
    name: http
    namespace: default
    spec:
    parentRefs:

    - name: gateway
      namespace: istio-ingress
      hostnames: ["httpbin.example.com"]
      rules:
    - matches: - path:
      type: PathPrefix
      value: /get
      backendRefs: - name: httpbin
      port: 8000
      EOF
      {{< /text >}}

1. Ingress ホスト環境変数を設定します：

   {{< text bash >}}
    $ kubectl wait -n istio-ingress --for=condition=programmed gateways.gateway.networking.k8s.io gateway
    $ export INGRESS_HOST=$(kubectl get gateways.gateway.networking.k8s.io gateway -n istio-ingress -ojsonpath='{.status.addresses[0].value}')
   {{< /text >}}

1. **curl** で `httpbin` サービスにアクセスします：

   {{< text bash >}}
    $ curl -s -I -HHost:httpbin.example.com "http://$INGRESS_HOST/get"
    ...
    HTTP/1.1 200 OK
    ...
    server: istio-envoy
    ...
   {{< /text >}}

   `-H` フラグで **Host** HTTP ヘッダーを "httpbin.example.com" に設定しています。
   これは `HTTPRoute` が "httpbin.example.com" のリクエストのみ処理するよう設定されているためです。
   テスト環境ではこのホスト名に DNS は割り当てられていないため、リクエストは Ingress IP に送信されます。

1. 明示的に公開していない URL にアクセスすると HTTP 404 エラーになります：

   {{< text bash >}}
    $ curl -s -I -HHost:httpbin.example.com "http://$INGRESS_HOST/headers"
    HTTP/1.1 404 Not Found
    ...
   {{< /text >}}

1. ルートを更新して `/headers` も公開し、リクエストヘッダーを追加します：

   {{< text bash >}}
    $ kubectl apply -f - <<EOF
    apiVersion: gateway.networking.k8s.io/v1
    kind: HTTPRoute
    metadata:
    name: http
    namespace: default
    spec:
    parentRefs:

    - name: gateway
      namespace: istio-ingress
      hostnames: ["httpbin.example.com"]
      rules:
    - matches: - path:
      type: PathPrefix
      value: /get - path:
      type: PathPrefix
      value: /headers
      filters: - type: RequestHeaderModifier
      requestHeaderModifier:
      add: - name: my-added-header
      value: added-value
      backendRefs: - name: httpbin
      port: 8000
      EOF
     {{< /text >}}

1. `/headers` に再度アクセスし、`My-Added-Header` ヘッダーが追加されていることを確認します：

   {{< text bash >}}
    $ curl -s -HHost:httpbin.example.com "http://$INGRESS_HOST/headers" | jq '.headers ["My-Added-Header"][0]'
    ...
    "added-value"
    ...
   {{< /text >}}

## デプロイ方法 {#deployment-methods}

上記の例では、ゲートウェイを構成する前に Ingress Gateway の `Deployment` をインストールする必要はありません。
デフォルト設定では `Gateway` 設定に基づき自動的に Gateway の `Deployment` と `Service` が作成されます。
高度なユースケースでは手動デプロイも可能です。

### 自動デプロイ {#automated-deployment}

デフォルトでは、各 `Gateway` ごとに `Service` と `Deployment` が自動生成されます。
それらは `<Gateway name>-<GatewayClass name>` という名前になります（`istio-waypoint` GatewayClass は除く）。
`Gateway` の設定が変更されると（例：新しいポート追加）、これらのリソースも自動更新されます。

`infrastructure` フィールドでこれらのリソースをカスタマイズできます：

{{< text yaml >}}
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
name: gateway
spec:
infrastructure:
annotations:
some-key: some-value
labels:
key: value
parametersRef:
group: ""
kind: ConfigMap
name: gw-options
gatewayClassName: istio
{{< /text >}}

`labels` や `annotations` の内容は生成リソースにコピーされます。
`parametersRef` で生成リソースを完全にカスタマイズできます。
これは `Gateway` と同じ名前空間の `ConfigMap` を参照する必要があります。

例：

{{< text yaml >}}
apiVersion: v1
kind: ConfigMap
metadata:
name: gw-options
data:
horizontalPodAutoscaler: |
spec:
minReplicas: 2
maxReplicas: 2

deployment: |
metadata:
annotations:
additional-annotation: some-value
spec:
replicas: 4
template:
spec:
containers: - name: istio-proxy
resources:
requests:
cpu: 1234m

service: |
spec:
ports: - "$patch": delete
port: 15021
{{< /text >}}

これらの設定は[戦略的マージパッチ](https://github.com/kubernetes/community/blob/master/contributors/devel/sig-api-machinery/strategic-merge-patch.md)方式で生成リソースに適用されます。
有効なキーは以下の通りです：

- `service`
- `deployment`
- `serviceAccount`
- `horizontalPodAutoscaler`
- `podDisruptionBudget`

{{< tip >}}
デフォルトでは `HorizontalPodAutoscaler` や `PodDisruptionBudget` は作成されませんが、カスタマイズで指定すれば作成されます。
{{< /tip >}}

#### GatewayClass のデフォルト値 {#gatewayclass-defaults}

各 `GatewayClass` ごとにすべての `Gateway` のデフォルト値を設定できます。
これは `gateway.istio.io/defaults-for-class: <gateway class name>` ラベル付きの `ConfigMap` で行います。
この `ConfigMap` は[ルート名前空間](/ja/docs/reference/config/istio.mesh.v1alpha1/#MeshConfig-root_namespace)（通常は `istio-system`）に配置します。各 `GatewayClass` につき 1 つのみ許可されます。
`ConfigMap` のフォーマットは `Gateway` のものと同じです。

このカスタマイズは `GatewayClass` と `Gateway` の両方に存在可能で、両方ある場合は `Gateway` 側が優先されます。

この `ConfigMap` はインストール時にも作成できます。例：

{{< text yaml >}}
kind: IstioOperator
spec:
values:
gatewayClasses:
istio:
deployment:
spec:
replicas: 2
{{< /text >}}

#### リソースのアタッチとスケーリング {#resource-attachment-and-scaling}

リソースは `Gateway` にアタッチしてカスタマイズできます。
ただし、現時点で多くの Kubernetes リソースは `Gateway` への直接アタッチをサポートしていません。
その場合は生成された `Deployment` や `Service` に直接アタッチします。
これらのリソースは[既知のラベル](https://gateway-api.sigs.k8s.io/geps/gep-1762/#resource-attachment)（`gateway.networking.k8s.io/gateway-name: <gateway name>`）や名前で識別できます：

- Gateway: `<gateway name>-<gateway class name>`
- waypoint: `<gateway name>`

例：`HorizontalPodAutoscaler` や `PodDisruptionBudget` を Gateway にアタッチする場合：

{{< text yaml >}}
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
name: gateway
spec:
gatewayClassName: istio
listeners:

- name: default
  hostname: "\*.example.com"
  port: 80
  protocol: HTTP
  allowedRoutes:
  namespaces:
  from: All

---

apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
name: gateway
spec:

# 生成された Deployment に参照を合わせる

# `kind: Gateway` ではなく `Deployment` を指定

scaleTargetRef:
apiVersion: apps/v1
kind: Deployment
name: gateway-istio
minReplicas: 2
maxReplicas: 5
metrics:

- type: Resource
  resource:
  name: cpu
  target:
  type: Utilization
  averageUtilization: 50

---

apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
name: gateway
spec:
minAvailable: 1
selector: # 生成された Deployment のラベルでマッチ
matchLabels:
gateway.networking.k8s.io/gateway-name: gateway
{{< /text >}}

### 手動デプロイ {#manual-deployment}

自動デプロイを使いたくない場合は、[手動で](/ja/docs/setup/additional-setup/gateway/) `Deployment` や `Service` を構成できます。

この場合、`Gateway` を `Service` に手動でリンクし、ポート設定も同期させる必要があります。

ポリシーアタッチ（例：AuthorizationPolicy の [`targetRef`](/ja/docs/reference/config/type/workload-selector/#PolicyTargetReference) フィールド利用）をサポートするには、Gateway Pod に `gateway.networking.k8s.io/gateway-name: <gateway name>` ラベルを追加してください。

`Gateway` を `Service` にリンクするには、`addresses` フィールドで**単一**の `Hostname` を指定します。

{{< tip >}}
`Service` と `Gateway` が異なる名前空間にある場合、Istio コントローラは `Service` を構成しません。
{{< /tip >}}

{{< text yaml >}}
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
name: gateway
spec:
addresses:

- value: ingress.istio-gateways.svc.cluster.local
  type: Hostname
  ...
  {{< /text >}}

## メッシュトラフィック {#mesh-traffic}

Gateway API でメッシュ内トラフィックも構成できます。
`parentRef` を Gateway ではなくサービスに向けて設定します。

例：クラスタ内の `example` サービスへの全リクエストにヘッダーを追加する場合：

{{< text yaml >}}
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
name: mesh
spec:
parentRefs:

- group: ""
  kind: Service
  name: example
  rules:
- filters: - type: RequestHeaderModifier
  requestHeaderModifier:
  add: - name: my-added-header
  value: added-value
  backendRefs: - name: example
  port: 80
  {{< /text >}}

詳細や他の例は[トラフィック管理](/ja/docs/tasks/traffic-management/)を参照してください。

## クリーンアップ {#cleanup}

1. `httpbin` サンプルとゲートウェイを削除します：

   {{< text bash >}}
    $ kubectl delete -f @samples/httpbin/httpbin.yaml@
    $ kubectl delete httproute http
    $ kubectl delete gateways.gateway.networking.k8s.io gateway -n istio-ingress
    $ kubectl delete ns istio-ingress
   {{< /text >}}

1. Istio をアンインストールします：

   {{< text bash >}}
    $ istioctl uninstall -y --purge
    $ kubectl delete ns istio-system
    $ kubectl delete ns istio-ingress
   {{< /text >}}

1. Gateway API CRD リソースが不要な場合は削除します：

   {{< text bash >}}
    $ kubectl kustomize "github.com/kubernetes-sigs/gateway-api/config/crd?ref={{< k8s_gateway_api_version >}}" | kubectl delete -f -
   {{< /text >}}
