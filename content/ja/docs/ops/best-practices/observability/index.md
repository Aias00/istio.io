---
title: オブザーバビリティのベストプラクティス
description: Istio でアプリケーションを観測する際のベストプラクティス。
force_inline_toc: true
weight: 50
owner: istio/wg-policies-and-telemetry-maintainers
test: n/a
---

## 本番規模の監視に Prometheus を使う {#using-Prometheus-for-production-scale-monitoring}

Istio および Prometheus で本番規模の監視を行う場合、[階層型フェデレーション](https://prometheus.io/docs/prometheus/latest/federation/#hierarchical-federation)と一連の[レコーディングルール](https://prometheus.io/docs/prometheus/latest/configuration/recording_rules/)の併用が推奨されます。

Istio のインストールでは [Prometheus](http://prometheus.io) はデフォルトでデプロイされませんが、[はじめに](/ja/docs/setup/getting-started/)の中で
[Prometheus 統合ガイド](/ja/docs/ops/integrations/prometheus/)の「オプション 1：クイックスタート」セクションに従って Prometheus のデプロイ手順が案内されています。
この Prometheus デプロイは意図的に短い保持ウィンドウ（6 時間）で構成されています。このクイックスタート Prometheus デプロイは、メッシュ上で動作するすべての Envoy
プロキシからメトリクスを収集し、それらのソースに関する一連のラベル（`instance`、`pod`、`namespace`）でメトリクスを拡張します。

{{< image width="80%"
    link="./production-prometheus.svg"
    alt="Prometheus を使った Istio 本番監視のアーキテクチャ。"
    caption="本番規模の Istio 監視"
    >}}

### レコーディングルールによるワークロードレベルの集約 {#workload-level-aggregation-via-recording-rules}

インスタンスや Pod レベルのメトリクスを集約するには、以下のレコーディングルールをデフォルトの Prometheus 設定に追加してください：

{{< tabset category-name="workload-metrics-aggregation" >}}

{{< tab name="Plain Prometheus Rules" category-value="prom-rules" >}}

{{< text yaml >}}
groups:

- name: "istio.recording-rules"
  interval: 5s
  rules:

  - record: "workload:istio_requests_total"
    expr: |
    sum without(instance, kubernetes_namespace, kubernetes_pod_name) (istio_requests_total)

  - record: "workload:istio_request_duration_milliseconds_count"
    expr: |
    sum without(instance, kubernetes_namespace, kubernetes_pod_name) (istio_request_duration_milliseconds_count)

  - record: "workload:istio_request_duration_milliseconds_sum"
    expr: |
    sum without(instance, kubernetes_namespace, kubernetes_pod_name) (istio_request_duration_milliseconds_sum)

  - record: "workload:istio_request_duration_milliseconds_bucket"
    expr: |
    sum without(instance, kubernetes_namespace, kubernetes_pod_name) (istio_request_duration_milliseconds_bucket)

  - record: "workload:istio_request_bytes_count"
    expr: |
    sum without(instance, kubernetes_namespace, kubernetes_pod_name) (istio_request_bytes_count)

  - record: "workload:istio_request_bytes_sum"
    expr: |
    sum without(instance, kubernetes_namespace, kubernetes_pod_name) (istio_request_bytes_sum)

  - record: "workload:istio_request_bytes_bucket"
    expr: |
    sum without(instance, kubernetes_namespace, kubernetes_pod_name) (istio_request_bytes_bucket)

  - record: "workload:istio_response_bytes_count"
    expr: |
    sum without(instance, kubernetes_namespace, kubernetes_pod_name) (istio_response_bytes_count)

  - record: "workload:istio_response_bytes_sum"
    expr: |
    sum without(instance, kubernetes_namespace, kubernetes_pod_name) (istio_response_bytes_sum)

  - record: "workload:istio_response_bytes_bucket"
    expr: |
    sum without(instance, kubernetes_namespace, kubernetes_pod_name) (istio_response_bytes_bucket)

  - record: "workload:istio_tcp_sent_bytes_total"
    expr: |
    sum without(instance, kubernetes_namespace, kubernetes_pod_name) (istio_tcp_sent_bytes_total)

  - record: "workload:istio_tcp_received_bytes_total"
    expr: |
    sum without(instance, kubernetes_namespace, kubernetes_pod_name) (istio_tcp_received_bytes_total)

  - record: "workload:istio_tcp_connections_opened_total"
    expr: |
    sum without(instance, kubernetes_namespace, kubernetes_pod_name) (istio_tcp_connections_opened_total)

  - record: "workload:istio_tcp_connections_closed_total"
    expr: |
    sum without(instance, kubernetes_namespace, kubernetes_pod_name) (istio_tcp_connections_closed_total)
    {{< /text >}}

{{< /tab >}}

{{< tab name="Prometheus Operator Rules CRD" category-value="prom-operator-rules" >}}

{{< text yaml >}}
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
name: istio-metrics-aggregation
labels:
app.kubernetes.io/name: istio-prometheus
spec:
groups:

- name: "istio.metricsAggregation-rules"
  interval: 5s
  rules: - record: "workload:istio_requests_total"
  expr: "sum without(instance, kubernetes_namespace, kubernetes_pod_name) (istio_requests_total)"

      - record: "workload:istio_request_duration_milliseconds_count"
        expr: "sum without(instance, kubernetes_namespace, kubernetes_pod_name) (istio_request_duration_milliseconds_count)"
      - record: "workload:istio_request_duration_milliseconds_sum"
        expr: "sum without(instance, kubernetes_namespace, kubernetes_pod_name) (istio_request_duration_milliseconds_sum)"
      - record: "workload:istio_request_duration_milliseconds_bucket"
        expr: "sum without(instance, kubernetes_namespace, kubernetes_pod_name) (istio_request_duration_milliseconds_bucket)"

      - record: "workload:istio_request_bytes_count"
        expr: "sum without(instance, kubernetes_namespace, kubernetes_pod_name) (istio_request_bytes_count)"
      - record: "workload:istio_request_bytes_sum"
        expr: "sum without(instance, kubernetes_namespace, kubernetes_pod_name) (istio_request_bytes_sum)"
      - record: "workload:istio_request_bytes_bucket"
        expr: "sum without(instance, kubernetes_namespace, kubernetes_pod_name) (istio_request_bytes_bucket)"

      - record: "workload:istio_response_bytes_count"
        expr: "sum without(instance, kubernetes_namespace, kubernetes_pod_name) (istio_response_bytes_count)"
      - record: "workload:istio_response_bytes_sum"
        expr: "sum without(instance, kubernetes_namespace, kubernetes_pod_name) (istio_response_bytes_sum)"
      - record: "workload:istio_response_bytes_bucket"
        expr: "sum without(instance, kubernetes_namespace, kubernetes_pod_name) (istio_response_bytes_bucket)"

      - record: "workload:istio_tcp_sent_bytes_total"
        expr: "sum without(instance, kubernetes_namespace, kubernetes_pod_name) (istio_tcp_sent_bytes_total)"
      - record: "workload:istio_tcp_received_bytes_total"
        expr: "sum without(instance, kubernetes_namespace, kubernetes_pod_name) (istio_tcp_received_bytes_total)"
      - record: "workload:istio_tcp_connections_opened_total"
        expr: "sum without(instance, kubernetes_namespace, kubernetes_pod_name) (istio_tcp_connections_opened_total)"
      - record: "workload:istio_tcp_connections_closed_total"
        expr: "sum without(instance, kubernetes_namespace, kubernetes_pod_name) (istio_tcp_connections_closed_total)"

  {{< /text >}}

{{< /tab >}}

{{< /tabset >}}

{{< tip >}}
上記のレコーディングルールは、Pod やインスタンスレベルのメトリクスを集約するだけです。これにより
[Istio 標準メトリクス](/ja/docs/reference/config/metrics/)のすべての項目と、すべての Istio 次元が保持されます。
フェデレーションでメトリクスの次元を制御するのに役立ちますが、既存のダッシュボードやアラート、特定の参照に合わせてレコーディングルールをさらに最適化したい場合もあるでしょう。

レコーディングルールの設定方法については、[レコーディングルールでメトリクス収集を最適化](/ja/docs/ops/best-practices/observability/#optimizing-metrics-collection-with-recording-rules)も参照してください。
{{< /tip >}}

### ワークロードレベル集約メトリクスを使ったフェデレーション {#federation-using-workload-level-aggregated-metrics}

Prometheus フェデレーションを構築するには、本番用 Prometheus デプロイの設定を変更し、Istio Prometheus フェデレーションエンドポイントからメトリクスを取得するようにします。

設定に以下のジョブを追加します：

{{< text yaml >}}

- job_name: 'istio-prometheus'
  honor_labels: true
  metrics_path: '/federate'
  kubernetes_sd_configs:
  - role: pod
    namespaces:
    names: ['istio-system']
    metric_relabel_configs:
  - source*labels: [__name__]
    regex: 'workload:(.*)'
    target*label: **name**
    action: replace
    params:
    'match[]': - '{**name**=~"workload:(.*)"}' - '{**name**=~"pilot(.\*)"}'
    {{< /text >}}

[Prometheus Operator](https://github.com/coreos/prometheus-operator) を使っている場合は、以下の設定を使います：

{{< text yaml >}}
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
name: istio-federation
labels:
app.kubernetes.io/name: istio-prometheus
spec:
namespaceSelector:
matchNames: - istio-system
selector:
matchLabels:
app: prometheus
endpoints:

- interval: 30s
  scrapeTimeout: 30s
  params:
  'match[]': - '{**name**=~"workload:(._)"}' - '{**name**=~"pilot(._)"}'
  path: /federate
  targetPort: 9090
  honorLabels: true
  metricRelabelings: - sourceLabels: ["__name__"]
  regex: 'workload:(.\*)'
  targetLabel: "**name**"
  action: replace
  {{< /text >}}

{{< tip >}}
フェデレーション設定のポイントは、まず Istio デプロイの Prometheus で収集した [Istio 標準メトリクス](/ja/docs/reference/config/metrics/) のジョブにマッチさせることです。
また、収集したメトリクスの名前からワークロードレベルのレコーディングルールのプレフィックス（`workload:`）を除去してリネームします。
これにより、既存のダッシュボードや参照が本番用 Prometheus でもシームレスに動作します（Istio インスタンスを直接参照しません）。

フェデレーション設定時に envoy や go など追加のメトリクスも含めることができます。

コントロールプレーンのメトリクスも本番用 Prometheus で収集・フェデレーションされます。
{{< /tip >}}

### レコーディングルールでメトリクス収集を最適化 {#optimizing-metrics-collection-with-recording-rules}

レコーディングルールで[Pod やインスタンスレベルの集約](#workload-level-aggregation-via-recording-rules)を行うだけでなく、既存のダッシュボードやアラート用に集約メトリクスを生成するためにレコーディングルールを使いたい場合もあるでしょう。このような収集の最適化は、本番用 Prometheus のリソース消費を大幅に削減し、参照パフォーマンスも向上します。

たとえば、ある監視ダッシュボードが以下の Prometheus クエリを使っているとします：

- 過去 1 分間のリクエストレートの平均値を宛先サービスと名前空間で集約

  {{< text plain >}}
  sum(irate(istio_requests_total{reporter="source"}[1m]))
  by (
  destination_canonical_service,
  destination_workload_namespace
  )
  {{< /text >}}

- 過去 1 分間の P95 クライアントレイテンシを送信元・宛先サービスと名前空間で集約

  {{< text plain >}}
  histogram_quantile(0.95,
  sum(irate(istio_request_duration_milliseconds_bucket{reporter="source"}[1m]))
  by (
  destination_canonical_service,
  destination_workload_namespace,
  source_canonical_service,
  source_workload_namespace,
  le
  )
  )
  {{< /text >}}

以下のレコーディングルールを Istio Prometheus 設定に追加し、`istio` プレフィックスを付けてフェデレーションでこれらのメトリクスを識別しやすくします。

{{< text yaml >}}
groups:

- name: "istio.recording-rules"
  interval: 5s
  rules:
  - record: "istio:istio_requests:by_destination_service:rate1m"
    expr: |
    sum(irate(istio_requests_total{reporter="destination"}[1m]))
    by (
    destination_canonical_service,
    destination_workload_namespace
    )
  - record: "istio:istio_request_duration_milliseconds_bucket:p95:rate1m"
    expr: |
    histogram_quantile(0.95,
    sum(irate(istio_request_duration_milliseconds_bucket{reporter="source"}[1m]))
    by (
    destination_canonical_service,
    destination_workload_namespace,
    source_canonical_service,
    source_workload_namespace,
    le
    )
    )
    {{< /text >}}

本番用 Prometheus インスタンスは、Istio インスタンスから以下のようにフェデレーション情報を取得できます：

- マッチ句 `{__name__=~"istio:(.*)"}`

- メトリクス名をリネーム：`regex: "istio:(.*)"`

元の参照は次のように置き換えられます：

- `istio_requests:by_destination_service:rate1m`

- `avg(istio_request_duration_milliseconds_bucket:p95:rate1m)`

{{< tip >}}
[AutoTrader の本番環境メトリクス収集最適化](https://karlstoney.com/2020/02/25/federated-prometheus-to-reduce-metric-cardinality/)に関する記事では、ダッシュボードやアラートのために参照集約を直接行う方法のより詳細な例が紹介されています。
{{< /tip >}}
