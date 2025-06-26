---
title: Envoy ベースのトレースの動作方法
weight: 11
---

Envoy ベースのトレース統合では、Envoy（サイドカー プロキシ）は、所代理のアプリケーションから直接トレース情報を後端サービスに送信します。

Envoy：

- リクエストを代理する際に、リクエスト ID とトレースヘッダー（例：`X-B3-TraceId`）を生成
- リクエストとレスポンスのメタデータ（即レスポンス時間）に基づいて、各リクエストのトレース範囲を生成
- 生成されたトレース範囲をトレース後端に送信
- トレースヘッダーを代理のアプリケーションに転送

Istio は [OpenTelemetry](/zh/docs/tasks/observability/distributed-tracing/opentelemetry/)
と互換性のあるバックエンドをサポートしています。
[Jaeger](/zh/docs/tasks/observability/distributed-tracing/jaeger/) もサポートしています。
他のサポートされているプラットフォームには [Zipkin](/zh/docs/tasks/observability/distributed-tracing/zipkin/)
と [Apache SkyWalking](/zh/docs/tasks/observability/distributed-tracing/skywalking/) があります。
