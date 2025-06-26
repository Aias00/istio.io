---
title: Istio での分散トレースの実装方法
weight: 0
---

Istio は [Envoy](#how-envoy-based-tracing-works) の分散トレースシステムを使用します。
[アプリケーションは、後続の送信リクエストのトレースヘッダー情報を転送する責任を負います](#istio-copy-headers)。

[分散トレースの概要](/zh/docs/tasks/observability/distributed-tracing/overview/)と
[Envoy トレースドキュメント](https://www.envoyproxy.io/docs/envoy/latest/intro/arch_overview/observability/tracing)で詳細を確認してください。
