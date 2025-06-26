---
title: なぜ私のリクエストがトレースされないのですか？
weight: 30
---

`default` [設定ファイル](/zh/docs/setup/additional-setup/config-profiles/)では、
トレースのサンプリング率は 1% に設定されています。
これは、Istio がトレース後端に報告する 100 個のトレースインスタンスのうち、1 個のみがキャプチャされることを意味します。
`demo` 設定ファイルでは、サンプリング率は 100% に設定されています。
サンプリング率の設定方法については、[このセクション](/zh/docs/tasks/observability/distributed-tracing/telemetry-api/#customizing-trace-sampling)を参照してください。

まだトレースデータが表示されない場合は、Istio [ポート命名規則](/zh/faq/traffic-management/#naming-port-convention)に従っているか確認してください。
また、サイドカー プロキシ (Envoy) がトラフィックをキャプチャできるように、適切なコンテナ ポートを公開していることを確認してください（例：pod spec を介して）。

エクスポート プロキシに関連するトレースデータのみが表示され、イングレスプロキシに関連するトレースデータが表示されない場合は、
Istio [ポート命名規則](/zh/about/faq/#naming-port-convention)に関連する可能性があります。
