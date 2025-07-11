---
title: 概要
description: Istio の分散トレースの概要。
weight: 1
keywords: [telemetry, tracing]
aliases:
  - /zh/docs/tasks/telemetry/distributed-tracing/overview/
owner: istio/wg-policies-and-telemetry-maintainers
test: n/a
---

分散トレースにより、ユーザーは複数の分散サービスメッシュをまたぐ 1 つのリクエストを追跡・分析できます。
これにより、リクエストのレイテンシ、シーケンス、並列性を可視化してより深く理解できます。

Istio は [Envoy の分散トレース](https://www.envoyproxy.io/docs/envoy/latest/intro/arch_overview/observability/tracing)機能を活用し、すぐに使えるトレース統合を提供します。

現在、ほとんどのトレースバックエンドは
[OpenTelemetry](/zh/docs/tasks/observability/distributed-tracing/opentelemetry/) プロトコルでトレースを受信できますが、
Istio は [Zipkin](/zh/docs/tasks/observability/distributed-tracing/zipkin/)
や [Apache SkyWalking](/zh/docs/tasks/observability/distributed-tracing/skywalking/) などの従来プロトコルもサポートしています。

## トレースの設定 {#configuring-tracing}

Istio は [Telemetry API](/zh/docs/tasks/observability/distributed-tracing/telemetry-api/) を提供しており、
分散トレースの設定（プロバイダーの選択、
[サンプリングレート](/zh/docs/tasks/observability/distributed-tracing/sampling/)の設定、ヘッダーの変更など）に利用できます。

## 拡張プロバイダー {#extension-providers}

[拡張プロバイダー](/zh/docs/reference/config/istio.mesh.v1alpha1/#MeshConfig-ExtensionProvider)は `MeshConfig` で定義され、
トレースバックエンドの設定を定義できます。サポートされているプロバイダーには OpenTelemetry、Zipkin、SkyWalking、Datadog、Stackdriver があります。

## トレースコンテキスト伝播をサポートするアプリケーションの構築 {#building-applications-to-support-trace-context-propagation}

Istio プロキシは自動的に span を送信できますが、これらの span を同じ呼び出しチェーンに関連付けるには追加情報が必要です。
そのため、プロキシが span 情報を送信する際、アプリケーションは適切な HTTP リクエストヘッダーを付与する必要があります。

これを実現するには、各アプリケーションが受信リクエストからヘッダーを収集し、
そのリクエストがトリガーするすべての送信リクエストにこれらのヘッダーを転送する必要があります。
どのヘッダーを転送するかは設定したトレースバックエンドによって異なり、
各トレースシステム固有のタスクページで説明されています。
以下はそのまとめです：

すべてのアプリケーションは以下のヘッダーを転送する必要があります：

- `x-request-id`：Envoy 固有のヘッダーで、ログやトレースの一貫したサンプリングに使用されます。
- `traceparent` および `tracestate`：[W3C 標準ヘッダー](https://www.w3.org/TR/trace-context/)

Zipkin の場合は [B3 マルチヘッダー形式](https://github.com/openzipkin/b3-propagation)を転送してください：

- `x-b3-traceid`
- `x-b3-spanid`
- `x-b3-parentspanid`
- `x-b3-sampled`
- `x-b3-flags`

商用オブザーバビリティツールについては、それぞれのドキュメントを参照してください。

例えば、[サンプル Python `productpage` サービス]({{< github_blob >}}/samples/bookinfo/src/productpage/productpage.py#L125) を見ると、
アプリケーションが OpenTelemetry ライブラリを使って HTTP リクエストからすべてのトレーサーに必要なヘッダーを抽出していることが分かります：

{{< text python >}}
def getForwardHeaders(request):
headers = {}

    # OpenTelemetry のスパンで x-b3-*** ヘッダーを埋めることができる
    ctx = propagator.extract(carrier={k.lower(): v for k, v in request.headers})
    propagator.inject(headers, ctx)

    # ...

        incoming_headers = ['x-request-id',
        'x-ot-span-context',
        'x-datadog-trace-id',
        'x-datadog-parent-id',
        'x-datadog-sampling-priority',
        'traceparent',
        'tracestate',
        'x-cloud-trace-context',
        'grpc-trace-bin',
        'user-agent',
        'cookie',
        'authorization',
        'jwt',
    ]

    # ...

    for ihdr in incoming_headers:
        val = request.headers.get(ihdr)
        if val is not None:
            headers[ihdr] = val

    return headers

{{< /text >}}

[reviews アプリケーション]({{< github_blob >}}/samples/bookinfo/src/reviews/reviews-application/src/main/java/application/rest/LibertyRestEndpoint.java#L186) (Java) では `requestHeaders` を使って同様の処理を行っています：

{{< text java >}}
@GET
@Path("/reviews/{productId}")
public Response bookReviewsById(@PathParam("productId") int productId, @Context HttpHeaders requestHeaders) {

// ...

if (ratings_enabled) {
JsonObject ratingsResponse = getRatings(Integer.toString(productId), requestHeaders);
{{< /text >}}

アプリケーションで下流呼び出しを行う際は、これらのヘッダーを必ず含めてください。
