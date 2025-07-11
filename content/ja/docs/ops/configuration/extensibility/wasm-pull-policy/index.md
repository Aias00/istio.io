---
title: WebAssembly モジュールのプルポリシー
description: Istio が Wasm モジュールをプルするかキャッシュバージョンを使うかをどのように決定するかを説明します。
weight: 10
keywords: [extensibility, Wasm, WebAssembly]
owner: istio/wg-policies-and-telemetry-maintainers
test: n/a
status: Alpha
---

{{< boilerplate alpha >}}

[WasmPlugin API](/ja/docs/reference/config/proxy_extensions/wasm-plugin) は、[Wasm モジュールをプロキシに配布する](/ja/docs/tasks/extensibility/wasm-module-distribution)方法を提供します。
各プロキシはリモートのイメージレジストリや HTTP サーバーから Wasm モジュールをプルするため、
Istio がどのような仕組みでモジュールのプルを選択するかを理解することは、可用性やパフォーマンスの観点で重要です。

## イメージプルポリシーと例外 {#image-pull-policy-and-exceptions}

Kubernetes の `ImagePullPolicy` と同様に、
[WasmPlugin](/ja/docs/reference/config/proxy_extensions/wasm-plugin/#WasmPlugin) にも `IfNotPresent` と `Always` の概念があり、
それぞれ「キャッシュされたモジュールを使う」「常にモジュールをプルする」ことを意味します。

ユーザーは `ImagePullPolicy` フィールドで Wasm モジュール取得の挙動を明示的に設定できます。
ただし、以下のケースでは Istio がユーザー指定の挙動を上書きすることがあります：

1. ユーザーが [WasmPlugin](/ja/docs/reference/config/proxy_extensions/wasm-plugin/#WasmPlugin) で `sha256` を指定した場合、`ImagePullPolicy` に関わらず `IfNotPresent` ポリシーが使われます。

1. `url` フィールドが OCI イメージを指し、その末尾にダイジェスト（例：
   `gcr.io/foo/bar@sha256:0123456789abcdef0123456789abcdef0123456789abcdef0123456789abcdef`）が付いている場合、
   `IfNotPresent` ポリシーが使われます。

リソースで `ImagePullPolicy` を指定しない場合、Istio のデフォルトは `IfNotPresent` です。
ただし、`url` フィールドが `latest` タグ付きの OCI イメージを指定している場合は、
Istio は `Always` 振る舞いを適用します。

## キャッシュされたモジュールのライフサイクル {#lifecycle-of-cached-modules}

各プロキシ（Sidecar プロキシやゲートウェイ）は Wasm モジュールをキャッシュします。
そのため、キャッシュされた Wasm モジュールの寿命は対応する Pod の寿命に依存します。
また、プロキシのメモリ使用量を最小限に抑えるための有効期限メカニズムもあります：
一定期間キャッシュされた Wasm モジュールが使われなければ、そのモジュールは削除されます。

この有効期限の挙動は [pilot-proxy](/ja/docs/reference/commands/pilot-agent/#envvars) の環境変数 `WASM_MODULE_EXPIRY` および `WASM_PURGE_INTERVAL` で設定できます。
有効期限の長さやチェック間隔を調整できます。

## 「Always」の意味 {#the-meaning-of-always}

Kubernetes では、`ImagePullPolicy: Always` は Pod 作成時に毎回イメージソースから直接イメージをプルすることを意味します。
新しい Pod が起動するたびに、Kubernetes は新しいイメージを再度プルします。

`WasmPlugin` の場合、`ImagePullPolicy: Always` は、対応する `WasmPlugin` Kubernetes リソースが作成または変更されるたびに、
Istio がイメージソースから直接モジュールをプルすることを意味します。
`Always` ポリシーを使う場合、`spec` や `metadata` の変更が Wasm モジュールのプルをトリガーします。
そのため、Pod のライフサイクルや単一プロキシのライフサイクル中に、イメージソースから複数回プルされる場合があります。
