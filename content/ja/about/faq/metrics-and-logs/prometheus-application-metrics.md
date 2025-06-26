---
title: Prometheus と Istio を使用してアプリケーション指標を取得できますか？
weight: 90
---

はい。[Prometheus](https://prometheus.io/) はオープンソースの監視システムと時系列データベースです。
Prometheus と Istio を組み合わせて、Istio とメッシュ内のアプリケーションの状態を記録し、追跡することができます。
[Grafana](/ja/docs/ops/integrations/grafana/) や [Kiali](/ja/docs/tasks/observability/kiali/) などのツールを使用して、指標を可視化できます。
[Prometheus の設定](/ja/docs/ops/integrations/prometheus/#Configuration)を参照して、指標の収集を有効にする方法を確認してください。

いくつかの注意事項：

- Prometheus Pod が istiod Pod によって生成された証明書を取得し、Prometheus に配布する前に起動した場合、
  Prometheus pod を再起動して、双方向 TLS 保護の対象情報を収集する必要があります。
- アプリケーションが専用ポートで Prometheus 指標を公開している場合、そのポートを Service と Deployment の仕様に追加する必要があります。
