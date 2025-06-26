---
title: Istio で分散トレースをサポートするには何が必要ですか？
weight: 10
---

Istio は、サービスメッシュ内でトレース範囲（Trace spans）を報告することをサポートしています。
しかし、異なるトレース範囲を統合して完全なトラフィック マップを取得するには、アプリケーションがトレース コンテキスト情報を受信および送信する必要があります。

具体的には、Istio はアプリケーションに Envoy が生成したリクエスト ID と標準ヘッダーを転送することを依存しています。
これらのヘッダーには次のものが含まれます：

- `x-request-id`
- `traceparent`
- `tracestate`

Zipkin ユーザーは、[B3 トレースヘッダーを伝播](/zh/docs/tasks/observability/distributed-tracing/overview/#trace-context-propagation)することを確認する必要があります。

- `x-b3-traceid`
- `x-b3-spanId`
- `x-b3-parentspanid`
- `x-b3-sampled`
- `x-b3-flags`
- `b3`

ヘッダーの伝播は、[OpenTelemetry](https://opentelemetry.io/docs/concepts/context-propagation/) などのクライアント ライブラリを使用して行うことができます。
また、[分散トレースのタスク](/zh/docs/tasks/observability/distributed-tracing/overview/#trace-context-propagation)で説明されているように、手動で行うこともできます。
