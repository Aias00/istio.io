---
title: コンポーネントのログ記録
description: 実行中のコンポーネントの動作を詳細に把握するためのコンポーネントレベルのログの使い方。
weight: 70
keywords: [ops]
aliases:
  - /zh/help/ops/component-logging
  - /zh/docs/ops/troubleshooting/component-logging
owner: istio/wg-user-experience-maintainers
test: no
---

Istio の各コンポーネントは柔軟なログフレームワークを利用しており、多くの機能や制御を提供して運用や診断を支援します。コンポーネント起動時にコマンドライン引数でログ記録の動作を制御できます。

## ログスコープ {#logging-scopes}

コンポーネントのログ出力は**スコープ**ごとに分類されます。スコープとは、制御可能な関連ログ情報のまとまりを指します。コンポーネントの機能によって、スコープの種類は異なります。すべてのコンポーネントには `default` スコープがあり、分類されないログ情報に使われます。

たとえば、現時点で `istioctl` には 25 個のスコープがあり、コマンドの各機能領域を表します：

- `ads`、`adsc`、`all`、`analysis`、`authn`、`authorization`、
  `ca`、`cache`、`cli`、`default`、`installer`、`klog`、`mcp`、
  `model`、`patch`、`processing`、`resource`、`source`、`spiffe`、
  `tpath`、`translator`、`util`、`validation`、`validationController`、
  `wle`

Pilot、Citadel、Galley にはそれぞれ独自のスコープがあり、詳細は[リファレンス](/ja/docs/reference/commands/)を参照してください。

各スコープには、次のいずれかの出力レベルが設定できます：

1. none
1. error
1. warn
1. info
1. debug

`none` は出力なし、`debug` は最も詳細な出力です。すべてのスコープのデフォルトレベルは `info` で、通常の Istio 利用時に十分なログ情報を提供します。

出力レベルはコマンドラインの `--log_output_level` オプションでも制御できます。例：

{{< text bash >}}
$ istioctl analyze --log_output_level klog:none,cli:info
{{< /text >}}

コマンドライン以外にも、[ControlZ](/ja/docs/ops/diagnostic-tools/controlz) インターフェースからも実行中コンポーネントの出力レベルを制御できます。

## 出力の制御 {#controlling-output}

ログは通常、コンポーネントの標準出力に送信されます。`--log_target` オプションで出力先を指定できます。ファイルシステムのパスや、標準出力・標準エラー出力を表す `stdout`・`stderr` など、カンマ区切りで複数指定可能です。

ログは通常、読みやすい形式で出力されます。`--log_as_json` オプションを使うと、出力を JSON 形式に強制でき、ツールによる処理が容易になります。

## ログローテーション {#log-rotation}

Istio コンポーネントはログのローテーションを自動管理でき、大きなログを小さなファイルに分割します。`--log_rotate` オプションでファイル名ベースのローテーションが可能です。派生名が個々のログファイルに使われます。

`--log_rotate_max_age` オプションでローテーション前の最大日数、`--log_rotate_max_size` で最大サイズ（MB 単位）、`--log_rotate_max_backups` で保持する最大ローテートファイル数を指定できます。古いファイルは自動削除されます。

## コンポーネントのデバッグ {#component-debugging}

`--log_caller` や `--log_stacktrace_level` オプションで、ログにプログラマ向け情報を含めるか制御できます。コンポーネントのエラー調査時に有用ですが、通常運用では不要です。
