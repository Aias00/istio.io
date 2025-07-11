---
title: 拡張性
description: Istio の WebAssembly プラグインシステムについて説明します。
weight: 50
keywords: [wasm, webassembly, emscripten, extension, plugin, filter]
owner: istio/wg-policies-and-telemetry-maintainers
test: n/a
---

WebAssembly は、Istio プロキシ（Envoy）の機能を拡張するためのサンドボックス技術です。
Proxy-Wasm サンドボックス API は、Mixer に代わる Istio の主要な拡張メカニズムです。

WebAssembly サンドボックスの目標：

- **効率性** - 低レイテンシ、低 CPU・メモリ消費の拡張メカニズムです。
- **機能性** - ポリシー実行、テレメトリ収集、ペイロード変換などが可能な拡張メカニズムです。
- **分離性** - 1 つのプラグイン内のプログラムのバグやクラッシュが他のプラグインに影響しません。
- **設定** - プラグインは他の Istio API と同様の API で設定され、動的な拡張設定が可能です。
- **運用** - 拡張はログのみ、フェイルオープン、フェイルクローズなどの形でアクセス・デプロイできます。
- **拡張開発者** - 複数のプログラミング言語で開発可能です。

この[講演動画](https://youtu.be/XdWmm_mtVXI)は WebAssembly 統合アーキテクチャの紹介です。

## ハイレベルアーキテクチャ {#high-level-architecture}

Istio 拡張（Proxy-Wasm プラグイン）はいくつかの構成要素から成ります：

- **フィルターサービスプロバイダーインターフェース（SPI）**：フィルター用 Proxy-Wasm プラグインの構築に使用
- **サンドボックス**：Envoy に組み込まれた V8 Wasm ランタイム
- **ホスト API**：リクエストヘッダー、トレーラー、メタデータの処理用
- **アウトバウンド API**：gRPC や HTTP リクエスト用
- **統計・ロギング API**：メトリクスや監視用

{{< image width="80%" link="./extending.svg" caption="Istio/Envoy の拡張" >}}

## 例 {#example}

[こちら](https://github.com/envoyproxy/envoy-wasm/tree/19b9fd9a22e27fcadf61a06bf6aac03b735418e6/examples/wasm)は、C++ でフィルター用 Proxy-Wasm プラグインを実装した例です。
[このガイド](https://github.com/istio-ecosystem/wasm-extensions/blob/master/doc/write-a-wasm-extension-with-cpp.md)に従って C++ で Wasm 拡張を実装できます。

## エコシステム {#ecosystem}

- [Istio エコシステム Wasm 拡張](https://github.com/istio-ecosystem/wasm-extensions)
- [Proxy-Wasm ABI 説明](https://github.com/proxy-wasm/spec)
- [Proxy-Wasm C++ SDK](https://github.com/proxy-wasm/proxy-wasm-cpp-sdk)
- [Proxy-Wasm Rust SDK](https://github.com/proxy-wasm/proxy-wasm-rust-sdk)
- [Proxy-Wasm AssemblyScript SDK](https://github.com/solo-io/proxy-runtime)
- [WebAssembly Hub](https://webassemblyhub.io/)
- [ネットワークプロキシの WebAssembly 拡張（動画）](https://www.youtube.com/watch?v=OIUPf8m7CGA)
