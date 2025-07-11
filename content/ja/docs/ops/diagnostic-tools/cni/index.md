---
title: Istio CNI プラグインのトラブルシューティング
description: Istio と CNI プラグインの診断に役立つツールと技術について説明します。
weight: 90
keywords: [debug, cni]
owner: istio/wg-networking-maintainers
test: n/a
---

このページでは、Istio CNI プラグインの問題解決方法を紹介します。この記事を読む前に、
[CNI のインストールと運用ガイド](/ja/docs/setup/additional-setup/cni/)を参照してください。

## ログ {#log}

Istio CNI プラグインのログは、`PodSpec` に基づいてアプリケーション Pod のトラフィックリダイレクトがどのように構成されたかを示します。

このプラグインはコンテナランタイムのプロセス空間で動作するため、`kubelet` のログに CNI のログエントリが出力されます。
デバッグを容易にするため、CNI プラグインは `istio-cni-node` DaemonSet にもログを送信します。

CNI プラグインのデフォルトログレベルは `info` です。より詳細なログを得るには、
`values.cni.logLevel` インストールオプションを編集し、CNI DaemonSet Pod を再起動してください。

Istio CNI DaemonSet Pod のログには、CNI プラグインのインストール状況や
[競合状態とその緩和策](/ja/docs/setup/additional-setup/cni/#race-condition--mitigation)に関する情報も含まれます。

## モニタリング {#monitoring}

CNI DaemonSet は[メトリクスを生成](/ja/docs/reference/commands/install-cni/#metrics)し、
CNI のインストール、準備完了、競合状態の緩和策を監視できます。デフォルトで Prometheus のアノテーション
（`prometheus.io/port`、`prometheus.io/path`）が `istio-cni-node` DaemonSet Pod に追加されます。
標準の Prometheus 設定でメトリクスを収集できます。

## DaemonSet の準備完了 {#daemonset-readiness}

CNI DaemonSet の準備完了は、Istio CNI プラグインが正しくインストール・構成されたことを示します。
Istio CNI DaemonSet が準備完了でない場合は問題が発生しています。`istio-cni-node` DaemonSet のログを確認してください。
また、`istio_cni_install_ready` メトリクスで CNI インストールの準備状況を追跡できます。

## 競合状態と緩和策 {#race-condition-repair}

Istio CNI DaemonSet ではデフォルトで[競合状態と緩和策](/ja/docs/setup/additional-setup/cni/#race-condition--mitigation)が有効になっており、
CNI プラグインが準備完了前に起動した Pod を強制的に削除します。どの Pod が削除されたかは、次のようなログ行で確認できます：

{{< text plain >}}
2021-07-21T08:32:17.362512Z info Deleting broken pod: service-graph00/svc00-0v1-95b5885bf-zhbzm
{{< /text >}}

また、`istio_cni_repair_pods_repaired_total` メトリクスで修復された Pod の数も追跡できます。

## Pod 起動失敗の診断 {#diagnose-pod-start-up-failure}

CNI プラグインでよくある問題は、コンテナネットワークのセットアップ失敗により Pod が起動できないことです。
通常、失敗の理由は Pod イベントに記録され、Pod の詳細情報で確認できます：

{{< text bash >}}
$ kubectl describe pod POD_NAME -n POD_NAMESPACE
{{< /text >}}

Pod で初期化エラーが継続する場合、Init コンテナ `istio-validation` のログに "connection refused" エラーがないか確認してください：

{{< text bash >}}
$ kubectl logs POD_NAME -n POD_NAMESPACE -c istio-validation
...
2021-07-20T05:30:17.111930Z error Error connecting to 127.0.0.6:15002: dial tcp 127.0.0.1:0->127.0.0.6:15002: connect: connection refused
2021-07-20T05:30:18.112503Z error Error connecting to 127.0.0.6:15002: dial tcp 127.0.0.1:0->127.0.0.6:15002: connect: connection refused
...
2021-07-20T05:30:22.111676Z error validation timeout
{{< /text >}}

`istio-validation` Init コンテナは、トラフィックリダイレクト先の入出力ポートをリッスンするローカル仮想サーバーをセットアップし、
テストトラフィックがこの仮想サーバーにリダイレクトできるかを検証します。CNI プラグインが Pod のトラフィックリダイレクトを正しく設定できていない場合、
`istio-validation` Init コンテナが Pod の起動をブロックし、トラフィックのバイパスを防ぎます。ネットワーク設定のエラーや想定外の動作がないか、`istio-cni-node` のログで Pod ID を検索してください。

CNI プラグインのもう一つの故障症状は、アプリケーション Pod が起動時に繰り返し削除されることです。
これはプラグインが正しくインストールされず、Pod のトラフィックリダイレクトを設定できない場合に発生します。
CNI の[競合状態と緩和策ロジック](/ja/docs/setup/additional-setup/cni/#race-condition--mitigation)は、競合状態による Pod の破損を検知し、Pod を連続して削除します。
この問題が発生した場合は、CNI DaemonSet のログでプラグインが正しくインストールされなかった理由を確認してください。
