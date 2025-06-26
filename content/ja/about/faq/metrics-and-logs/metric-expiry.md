---
title: 短期指標を管理するにはどうすればよいですか？
weight: 20
---

短期指標は Prometheus のパフォーマンスを妨げる可能性があります。
これらは通常、ラベルの基数の重要なソースです。
基数はラベルの一意の値の数を測定する指標です。
短期指標が Prometheus に与える影響を管理するには、まず高基数指標とラベルを特定する必要があります。
Prometheus は `/status` ページで基数情報を提供します。
[PromQL](https://www.robustperception.io/which-are-my-biggest-metrics) を使用して、他の情報を取得できます。
Istio 指標の基数を削減する方法はいくつかあります：

* ホスト ヘッダーのフォールバックを無効にします。
  `destination_service` ラベルは高基数の潜在的なソースです。
  Istio プロキシが他のリクエスト メタデータからターゲット サービスを特定できない場合、`destination_service` の値はデフォルトでホスト ヘッダーに表示されます。
  クライアントがさまざまなホスト ヘッダーを使用する場合、これは `destination_service` の値が大量に生成される可能性があることを意味します。
  この場合、[指標のカスタマイズ](/ja/docs/tasks/observability/metrics/customize-metrics/)ガイドに従って、ホスト ヘッダーのフォールバックを無効にしてください。
  特定のワークロードまたは名前空間のホスト ヘッダーのフォールバックを無効にするには、`EnvoyFilter` 設定をコピーし、ホスト ヘッダーのフォールバックを無効にし、より具体的なセレクターを適用する必要があります。
  [この問題](https://github.com/istio/istio/issues/25963#issuecomment-666037411)には、これを実現する方法の詳細が含まれています。
* 不要なラベルをセットから削除します。高基数のラベルが不要な場合、`tags_to_remove` を使用して指標セットから削除できます。
* ラベル値を正規化するには、[Prometheus フェデレーション](/ja/docs/ops/best-practices/observability/#using-prometheus-for-production-scale-monitoring)または[リクエストの分類](/ja/docs/tasks/observability/metrics/classify-metrics/)を使用します。
  ラベルによって提供される情報が必要な場合、[Prometheus フェデレーション](/ja/docs/ops/best-practices/observability/#using-prometheus-for-production-scale-monitoring)または[リクエストの分類](/ja/docs/tasks/observability/metrics/classify-metrics/)を使用します。
