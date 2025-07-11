---
title: 可観測性
description: Istio が提供するテレメトリーとモニタリング機能について説明します。
weight: 40
keywords: [telemetry, metrics, logs, tracing]
aliases:
  - /zh/docs/concepts/policy-and-control/mixer.html
  - /zh/docs/concepts/policy-and-control/mixer-config.html
  - /zh/docs/concepts/policy-and-control/attributes.html
  - /zh/docs/concepts/policies-and-telemetry/overview/
  - /zh/docs/concepts/policies-and-telemetry/config/
  - /zh/docs/concepts/policies-and-telemetry/
owner: istio/wg-policies-and-telemetry-maintainers
test: n/a
---

Istio はメッシュ内のすべてのサービス通信に対して詳細なテレメトリーデータを生成します。このテレメトリーによりサービスの**可観測性**が得られ、運用者はサービス開発者に追加の負担をかけることなく、トラブルシューティング・保守・最適化を行うことができます。Istio を使うことで、監視対象サービスが他のサービスや Istio コンポーネントとどのように連携しているかを包括的に把握できます。

Istio はサービスメッシュ全体の可観測性を提供するため、以下の種類のテレメトリーデータを生成します：

- [**メトリクス**](#metrics)：Istio は監視のゴールデンシグナル（レイテンシ、トラフィック、エラー、飽和）に基づく一連のサービスメトリクスを生成します。
  Istio は[メッシュコントロールプレーン](/ja/docs/ops/deployment/architecture/)向けにも詳細なメトリクスを提供します。
  さらに、これらのメトリクスに基づくデフォルトのメッシュ監視ダッシュボードも提供されます。
- [**分散トレーシング**](#distributed-traces)：Istio は各サービスに分散トレース span を生成し、運用者がメッシュ内サービスの依存関係や呼び出しフローを理解できるようにします。
- [**アクセスログ**](#access-logs)：メッシュ内サービスへのトラフィック時、Istio は各リクエストの完全な記録（ソース・宛先のメタデータ含む）を生成できます。
  この情報により、運用者はサービス動作の監査を単一の[ワークロードインスタンス](/ja/docs/reference/glossary/#workload-instance)レベルまで制御できます。

## メトリクス {#metrics}

メトリクスは、集約的な方法で動作を監視・理解する手段を提供します。

サービス動作を監視するため、Istio はサービスメッシュ内のすべての入出トラフィックおよびメッシュ内部のサービス間トラフィックに対してメトリクスを生成します。これらのメトリクスは、総トラフィック数・エラー率・リクエスト応答時間などの動作情報を提供します。

メッシュ内サービスの動作監視だけでなく、メッシュ自体の動作監視も重要です。
Istio コンポーネントは自身の内部動作メトリクスもエクスポートでき、メッシュコントロールプレーンの機能や健全性の洞察を提供します。

### プロキシレベルのメトリクス {#proxy-level-metrics}

Istio のメトリクス収集は Sidecar プロキシ（Envoy）から始まります。
各プロキシは通過するすべてのトラフィック（インバウンド・アウトバウンド）に対して豊富なメトリクスを生成します。
プロキシはまた、自身の管理機能に関する詳細な統計情報（設定やヘルス情報など）も提供します。

Envoy が生成するメトリクスは、リスナーやクラスタなどのリソース単位でメッシュ監視を可能にします。
そのため、Envoy メトリクスを監視するには、メッシュサービスと Envoy リソースの関係を理解する必要があります。

Istio では、各ワークロードインスタンスごとにどの Envoy メトリクスを生成・収集するか選択できます。
デフォルトでは、Istio は Envoy が生成する統計のごく一部のみをサポートし、
過剰なバックエンド依存やメトリクス収集による CPU オーバーヘッドを抑えています。
ただし、必要に応じてプロキシメトリクスの収集範囲を簡単に拡張できます。
これにより、ネットワーク動作のピンポイントなデバッグと、メッシュ全体の監視コスト低減を両立できます。

[Envoy ドキュメント](https://www.envoyproxy.io/docs/envoy/latest/)には、
[Envoy 統計情報収集](https://www.envoyproxy.io/docs/envoy/latest/intro/arch_overview/observability/statistics.html?highlight=statistics)の詳細な説明があります。
[Envoy 統計](/ja/docs/ops/configuration/telemetry/envoy-stats/)の運用ガイドでは、プロキシレベルのメトリクス生成制御についてさらに詳しく解説しています。

プロキシレベルメトリクスの例：

{{< text json >}}
envoy_cluster_internal_upstream_rq{response_code_class="2xx",cluster_name="xds-grpc"} 7163

envoy_cluster_upstream_rq_completed{cluster_name="xds-grpc"} 7164

envoy_cluster_ssl_connection_error{cluster_name="xds-grpc"} 0

envoy_cluster_lb_subsets_removed{cluster_name="xds-grpc"} 0

envoy_cluster_internal_upstream_rq{response_code="503",cluster_name="xds-grpc"} 1
{{< /text >}}

### サービスレベルのメトリクス {#service-level-metrics}

プロキシレベルメトリクスに加え、Istio はサービス間通信を監視するためのサービス指向メトリクスも提供します。
これらのメトリクスは、サービス監視の 4 つの基本要素（レイテンシ、トラフィック、エラー、飽和）をカバーします。
Istio には、これらのメトリクスに基づくサービス動作監視用の[ダッシュボード](/ja/docs/tasks/observability/metrics/using-istio-dashboard/)がデフォルトで用意されています。

デフォルトでは、[標準 Istio メトリクス](/ja/docs/reference/config/metrics/)は
[Prometheus](/ja/docs/ops/integrations/prometheus/)にエクスポートされます。

サービスレベルメトリクスの利用は完全にオプションです。運用者は必要に応じてメトリクス生成・収集を無効化できます。

サービスレベルメトリクスの例：

{{< text json >}}
istio_requests_total{
connection_security_policy="mutual_tls",
destination_app="details",
destination_canonical_service="details",
destination_canonical_revision="v1",
destination_principal="cluster.local/ns/default/sa/default",
destination_service="details.default.svc.cluster.local",
destination_service_name="details",
destination_service_namespace="default",
destination_version="v1",
destination_workload="details-v1",
destination_workload_namespace="default",
reporter="destination",
request_protocol="http",
response_code="200",
response_flags="-",
source_app="productpage",
source_canonical_service="productpage",
source_canonical_revision="v1",
source_principal="cluster.local/ns/default/sa/default",
source_version="v1",
source_workload="productpage-v1",
source_workload_namespace="default"
} 214
{{< /text >}}

### コントロールプレーンメトリクス {#control-plane-metrics}

Istio コントロールプレーンも自己監視用のメトリクスを提供します。これらのメトリクスにより、Istio 自身の動作（メッシュ内サービスとは異なる）を監視できます。

これらのメトリクスの詳細は[リファレンス](/ja/docs/reference/commands/pilot-discovery/#metrics)を参照してください。

## 分散トレーシング {#distributed-traces}

分散トレーシングは、メッシュを流れる個々のリクエストを追跡することで、動作の監視と理解を可能にします。
トレーシングにより、運用者はサービスの依存関係やメッシュ内のレイテンシ発生源を把握できます。

Istio は Envoy プロキシ経由で分散トレーシングをサポートします。プロキシはアプリケーションが適切なリクエストコンテキストを転送するだけで、自動的にトレース span を生成します。

Istio は多くのトレーシングシステムをサポートしており、[Zipkin](/ja/docs/tasks/observability/distributed-tracing/zipkin/)、[Jaeger](/ja/docs/tasks/observability/distributed-tracing/jaeger/)、[OpenTelemetry](/ja/docs/tasks/observability/distributed-tracing/opentelemetry/) 対応の各種ツール・サービスが利用できます。
オペレーターはリンク生成のサンプリングレート（リクエストごとにトレースデータを生成する割合）を制御できます。
これにより、メッシュで生成されるトレースデータの量や頻度を調整できます。

Istio の分散トレーシングの詳細は[分散トレーシング FAQ](/ja/about/faq/distributed-tracing/)を参照してください。

Istio が単一リクエストに対して生成する分散トレース例：

{{< image link="/ja/docs/tasks/observability/distributed-tracing/zipkin/istio-tracing-details-zipkin.png" caption="単一リクエストの分散トレース" >}}

## アクセスログ {#access-logs}

アクセスログは、単一ワークロードインスタンスの視点から動作を監視・理解する手段を提供します。

Istio はサービストラフィックに対して、運用者がログの記録方法・内容・タイミング・出力先を完全に制御できる一連のフォーマットでアクセスログを生成できます。
詳細は[Envoy のアクセスログ取得](/ja/docs/tasks/observability/logs/access-log/)を参照してください。

Istio アクセスログの例：

{{< text plain >}}
[2019-03-06T09:31:27.360Z] "GET /status/418 HTTP/1.1" 418 - "-" 0 135 5 2 "-" "curl/7.60.0" "d209e46f-9ed5-9b61-bbdd-43e22662702a" "httpbin:8000" "127.0.0.1:80" inbound|8000|http|httpbin.default.svc.cluster.local - 172.30.146.73:80 172.30.146.82:38618 outbound*.8000*.\_.httpbin.default.svc.cluster.local
{{< /text >}}
