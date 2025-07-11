---
title: WebAssembly モジュール配布
description: メッシュ内でリモート WebAssembly モジュールを使用する方法を説明します。
weight: 10
aliases:
  - /zh/help/ops/extensibility/distribute-remote-wasm-module
  - /zh/docs/ops/extensibility/distribute-remote-wasm-module
  - /zh/ops/configuration/extensibility/wasm-module-distribution
keywords: [extensibility, Wasm, WebAssembly]
owner: istio/wg-policies-and-telemetry-maintainers
test: yes
status: Alpha
---

{{< boilerplate alpha >}}

Istio は[WebAssembly（Wasm）を使用してプロキシ機能を拡張する](/zh/blog/2020/wasm-announce/)機能を提供します。
Wasm 拡張性の主な利点の一つは、拡張機能をランタイムで動的にロードできることです。
これらの拡張機能は、まず Envoy プロキシに配布される必要があります。
Istio は、プロキシが Wasm モジュールを動的にダウンロードできるようにすることで、これを実現しています。

## テストアプリケーションのインストール{#setup-the-test-application}

このタスクを始める前に、[Bookinfo](/zh/docs/examples/bookinfo/#deploying-the-application)サンプルアプリケーションをデプロイしてください。

## Wasm モジュールの設定{#configure-wasm-modules}

この例では、メッシュ内に HTTP Basic 認証拡張機能を追加します。
Istio を設定して、リモートイメージレジストリから[Basic 認証モジュール](https://github.com/istio-ecosystem/wasm-extensions/tree/master/extensions/basic_auth)を取得しロードします。
このモジュールは、`/productpage` への呼び出し時に動作するように設定されます。

リモート Wasm モジュールを持つ WebAssembly フィルターを設定するには、
`WasmPlugin` リソースを作成します：

{{< text bash >}}
$ kubectl apply -f - <<EOF
apiVersion: extensions.istio.io/v1alpha1
kind: WasmPlugin
metadata:
name: basic-auth
namespace: istio-system
spec:
selector:
matchLabels:
istio: ingressgateway
url: oci://ghcr.io/istio-ecosystem/wasm-extensions/basic_auth:1.12.0
phase: AUTHN
pluginConfig:
basic_auth_rules: - prefix: "/productpage"
request_methods: - "GET" - "POST"
credentials: - "ok:test" - "YWRtaW4zOmFkbWluMw=="
EOF
{{< /text >}}

HTTP フィルターは認証フィルターとしてイングレスゲートウェイプロキシに注入されます。
Istio プロキシは `WasmPlugin` 設定を解釈し、OCI イメージレジストリからリモート
Wasm モジュールをローカルファイルにダウンロードし、そのファイルを参照して HTTP フィルターを Envoy に注入します。

{{< idea >}}
`istio-system` 以外の特定の名前空間で `WasmPlugin` を作成した場合、その名前空間内の Pod のみが設定されます。`istio-system` 名前空間でリソースを作成した場合、すべての名前空間に影響します。
{{< /idea >}}

## 設定済み Wasm モジュールの確認{#check-the-configured-wasm-module}

1. 資格情報なしで `/productpage` をテスト

   {{< text bash >}}
   $ curl -s -o /dev/null -w "%{http_code}" "http://$INGRESS_HOST:$INGRESS_PORT/productpage"
   401
   {{< /text >}}

1. 資格情報付きで `/productpage` をテスト

   {{< text bash >}}
   $ curl -s -o /dev/null -w "%{http_code}" -H "Authorization: Basic YWRtaW4zOmFkbWluMw==" "http://$INGRESS_HOST:$INGRESS_PORT/productpage"
   200
   {{< /text >}}

`WasmPlugin` API のさらなる使用例については、
[API リファレンス](/zh/docs/reference/config/proxy_extensions/wasm-plugin/)をご覧ください。

## Wasm モジュールのクリーンアップ{#clean-up-wasm-modules}

{{< text bash >}}
$ kubectl delete wasmplugins.extensions.istio.io -n istio-system basic-auth
{{< /text >}}

## Wasm モジュール配布の監視{#monitor-wasm-module-distribution}

リモート Wasm モジュールの配布状況を追跡するためのいくつかの統計情報があります。

Istio プロキシは以下の統計情報を収集します：

- `istio_agent_wasm_cache_lookup_count`: Wasm リモート取得キャッシュの検索回数。
- `istio_agent_wasm_cache_entries`: Wasm 設定変換と結果の数。
  成功、リモートロードなし、マーシャリング失敗、リモート取得失敗、リモート取得ヒント未受信を含みます。
- `istio_agent_wasm_config_conversion_duration_bucket`: istio-agent が Wasm モジュールの設定変換に費やした合計時間（ミリ秒単位）。
- `istio_agent_wasm_remote_fetch_count`: Wasm リモート取得とその結果の数。
  成功、ダウンロード失敗、チェックサム不一致を含みます。

ダウンロード失敗などで Wasm フィルター設定が拒否された場合、
istiod も `type.googleapis.com/envoy.config.core.v3.TypedExtensionConfig` タイプラベル付きの `pilot_total_xds_rejects` を発行します。

## Wasm 拡張機能の開発{#develop-a-wasm-extension}

Wasm モジュール開発の詳細については、
Istio コミュニティが管理する [`istio-ecosystem/wasm-extensions`](https://github.com/istio-ecosystem/wasm-extensions) リポジトリのガイドを参照してください。
このリポジトリは Istio の Telemetry Wasm 拡張機能の開発に使用されます：

- [C++ で Wasm 拡張機能を作成、テスト、デプロイ、メンテナンスする方法](https://github.com/istio-ecosystem/wasm-extensions/blob/master/doc/write-a-wasm-extension-with-cpp.md)
- [Istio Wasm プラグイン互換の OCI イメージをビルドする方法](https://github.com/istio-ecosystem/wasm-extensions/blob/master/doc/how-to-build-oci-images.md)
- [C++ Wasm 拡張機能のユニットテストを書く方法](https://github.com/istio-ecosystem/wasm-extensions/blob/master/doc/write-cpp-unit-test.md)
- [Wasm 拡張機能の統合テストを書く方法](https://github.com/istio-ecosystem/wasm-extensions/blob/master/doc/write-integration-test.md)

## 制限事項{#limitations}

このモジュールの配布メカニズムにはいくつか既知の制限があり、今後のリリースで解決される予定です：

- HTTP フィルターのみサポートされています。
