---
title: 既存の Mixer 機能を移行するにはどうすればよいですか？
weight: 30
---

Mixer は [Istio 1.8 バージョンで削除されました](/ja/news/releases/1.8.x/announcing-1.8/#deprecations)。
Mixer の組み込みアダプターまたはメッシュ拡張に依存している場合は、移行が必要です。

組み込みアダプターの場合、いくつかの代替ソリューションが提供されています：

* `Prometheus` と `Stackdriver` の統合は[プロキシ拡張](/ja/docs/reference/config/proxy_extensions/)として実装されています。
  これらの拡張機能によって生成されるテレメトリのカスタマイズは、[リクエストの分類](/ja/docs/tasks/observability/metrics/classify-metrics/)と[Prometheus 指標のカスタマイズ](/ja/docs/tasks/observability/metrics/customize-metrics/)を使用して実現できます。
* グローバルおよびローカルのレート制限（`memquota` および `redisquota` アダプター）
  機能は[Envoy ベースのレート制限ソリューション](/ja/docs/tasks/policy-enforcement/rate
  このソリューションは [OPA ポリシー プロキシとの統合](https://www.openpolicyagent.org/docs/latest/envoy-introduction/)をサポートしています。

カスタム プロセス外アダプターの場合、Wasm ベースの拡張機能に移行することを強くお勧めします。
[Wasm モジュールの開発](https://github.com/istio-ecosystem/wasm-extensions/blob/master/doc/write-a-wasm-extension-with-cpp.md)と[拡張の配布](/ja/docs/tasks/extensibility/wasm-module-distribution/)に関するガイドを参照してください。

一時的なソリューションとして、[Mixer で Envoy ext-authz と gRPC アクセス ログ API のサポートを有効にする](https://github.com/istio/istio/wiki/Enabling-Envoy-Authorization-Service-and-gRPC-Access-Log-Service-With-Mixer)を使用して、
Istio を 1.7 バージョンにアップグレードし、1.7 Mixer のプロセス外アダプターを使用したままにすることができます。
これにより、Wasm ベースの拡張機能に移行するための時間が増えます。
この一時的なソリューションはテストされていないことに注意してください。
Istio 1.7 ブランチでのみ利用可能であり、2021 年 2 月 以降のサポート ウィンドウ外であるため、パッチ修正が困難です。
