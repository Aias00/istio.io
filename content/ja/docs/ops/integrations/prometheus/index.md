---
title: Prometheus
description: Prometheus との統合方法。
weight: 30
keywords: [integration, prometheus]
owner: istio/wg-environments-maintainers
test: n/a
---

[Prometheus](https://prometheus.io/) はオープンソースの監視システムおよび時系列データベースです。Prometheus を Istio と統合することで、メトリクスを収集し、Istio やメッシュ内アプリケーションの稼働状況を把握できます。[Grafana](/ja/docs/ops/integrations/grafana/) や [Kiali](/ja/docs/tasks/observability/kiali/) でこれらのメトリクスを可視化できます。

## インストール {#installation}

### オプション 1：クイックスタート {#option-1-quick-start}

Istio では Prometheus を素早く起動できる簡単なインストール例を提供しています：

{{< text bash >}}
$ kubectl apply -f {{< github_file >}}/samples/addons/prometheus.yaml
{{< /text >}}

このコマンドで Prometheus がクラスタにデプロイされます。これはデモ用であり、パフォーマンスやセキュリティの調整はされていません。

{{< warning >}}
クイックスタートの構成は小規模クラスタや短期監視向けであり、大規模メッシュや長期監視には適していません。特に、ラベルの追加はメトリクスのカーディナリティを増やし、多くのメモリを必要とします。また、トラフィックの傾向を把握するには履歴データが必要です。
{{< /warning >}}

### オプション 2：カスタムインストール {#option-2-customizable-install}

[Prometheus ドキュメント](https://www.prometheus.io/)を参照して、環境に合わせて Prometheus をインストール・デプロイしてください。[設定](#configuration)も参照し、より多くの Istio メトリクスを収集するための設定方法を確認してください。

## 設定 {#configuration}

Istio メッシュ内の各コンポーネントは、外部にメトリクスを公開するエンドポイントを持っています。Prometheus はこれらのエンドポイントからメトリクスを収集します。

[Prometheus 設定ファイル](https://prometheus.io/docs/prometheus/latest/configuration/configuration/)で、収集対象のエンドポイントやポート、パス、TLS 設定などを制御できます。

メッシュ全体のメトリクスを収集するには、Prometheus で以下を設定してください：

1. コントロールプレーン（`istiod` Deployment）
1. イングレス・エグレスゲートウェイ
1. Envoy Sidecar
1. ユーザーアプリケーション（アプリが Prometheus メトリクスを公開している場合）

メトリクス設定を簡素化するため、Istio では 2 つの運用モードを提供しています：

### オプション 1：メトリクスのマージ {#option-1-metrics-merging}

Istio では `prometheus.io` アノテーションでメトリクス収集を制御できます。これにより、Helm の `stable/prometheus` Chart など、標準的な設定でそのままメトリクス収集が可能です。

{{< tip >}}
`prometheus.io` は Prometheus のコアアノテーションではありませんが、メトリクス収集の標準的なアノテーションとして広く使われています。
{{< /tip >}}

このオプションはデフォルトで有効ですが、[インストール](/ja/docs/setup/install/istioctl/)時に `--set meshConfig.enablePrometheusMerge=false` で無効化できます。有効時は、すべてのデータプレーンコンテナに適切な `prometheus.io` アノテーションが追加され、メトリクス収集が設定されます。既存のアノテーションがある場合は上書きされます。このオプションでは、Envoy Sidecar が Istio のメトリクスとアプリケーションのメトリクスをマージします。マージされたメトリクスは `:15020/stats/prometheus` で収集されます。

このオプションでは、すべてのメトリクスがプレーンテキストで表示されます。

以下の場合はこのオプションが適しません：

- TLS でメトリクスを収集したい場合
- アプリケーションが Istio と同名のメトリクスを公開している場合（例：`istio_requests_total` など）。アプリが Envoy を実行している場合に発生することがあります。
- Prometheus Deployment が `prometheus.io` アノテーションによる収集を設定していない場合

必要に応じて、Pod に `prometheus.istio.io/merge-metrics: "false"` を追加してこの機能を無効化できます。

### オプション 2：カスタム収集設定 {#option-2-customized-scraping-configurations}

既存の Prometheus 設定で Istio のメトリクスを収集するには、いくつかの Job を追加する必要があります。

- `Istiod` の状態を取得するには、`http-monitoring` ポートを収集する以下の例を追加します：

{{< text yaml >}}

- job_name: 'istiod'
  kubernetes_sd_configs:
  - role: endpoints
    namespaces:
    names: - istio-system
    relabel_configs:
  - source_labels: [__meta_kubernetes_service_name, __meta_kubernetes_endpoint_port_name]
    action: keep
    regex: istiod;http-monitoring
    {{< /text >}}

* Envoy の状態（Sidecar やゲートウェイのプロキシ）を収集するには、`-envoy-prom` ポートを収集する以下の Job を追加します：

{{< text yaml >}} - job_name: 'envoy-stats'
metrics_path: /stats/prometheus
kubernetes_sd_configs: - role: pod

      relabel_configs:
      - source_labels: [__meta_kubernetes_pod_container_port_name]
        action: keep
        regex: '.*-envoy-prom'

{{< /text >}}

- アプリケーションの状態については、[Strict mTLS](/ja/docs/tasks/security/authentication/authn-policy/#globally-enabling-istio-mutual-tls-in-strict-mode) を無効にしていれば既存の設定で収集できます。そうでない場合は、Prometheus を[Istio 証明書で収集](#tls-settings)するよう設定が必要です。

#### TLS 設定 {#tls-settings}

コントロールプレーン、ゲートウェイ、Envoy Sidecar のメトリクスはプレーンテキストで収集されますが、アプリケーションのメトリクスはワークロードに設定された[Istio 認証ポリシー](/ja/docs/tasks/security/authentication/authn-policy)に従います。

- `STRICT` モードの場合、Prometheus を Istio 証明書で収集するよう設定が必要です。
- `PERMISSIVE` モードの場合、ワークロードは通常 TLS とプレーンテキストの両方を受け入れます。ただし、Prometheus は `PERMISSIVE` モードで必要な TLS バリアントを送信できません。そのため、Prometheus では TLS 設定を行わないでください。
- `DISABLE` モードの場合、Prometheus で TLS 設定は不要です。

{{< tip >}}
これは Istio が終端する TLS の場合のみ該当します。アプリケーションが直接 TLS を処理する場合：

- `STRICT` モードはサポートされません。Prometheus で 2 重 TLS を送信する必要がありますが、これはできません。
- `PERMISSIVE` モードと `DISABLE` モードは、Istio が存在しない場合と同様に設定してください。

詳細は[TLS 設定の理解](/ja/docs/ops/configuration/traffic-management/tls-configuration/)を参照してください。
{{< /tip >}}

Prometheus で Istio 証明書を利用するもう 1 つの方法は Sidecar です。Sidecar は SDS 証明書を転送し、Prometheus と共有できるボリュームに出力します。ただし、Sidecar で Prometheus のリクエストをインターセプトしてはいけません。Prometheus のポートのアクセスモードは Istio Sidecar プロキシモデルと互換性がありません。

このため、Prometheus サーバーコンテナで証明書ボリュームをマウントします：

{{< text yaml >}}
containers:

- name: prometheus-server
  ...
  volumeMounts:
  mountPath: /etc/prom-certs/
  name: istio-certs
  volumes:
- emptyDir:
  medium: Memory
  name: istio-certs
  {{< /text >}}

次に、Prometheus Deployment の Pod Template に以下のアノテーションを追加し、[Sidecar インジェクション](/ja/docs/setup/additional-setup/sidecar-injection/)を利用します。これにより、Sidecar が共有ボリュームに証明書を書き込みますが、トラフィックリダイレクトは設定されません。

{{< text yaml >}}
spec:
template:
metadata:
annotations:
traffic.sidecar.istio.io/includeInboundPorts: "" # 入口トラフィックをインターセプトしない
traffic.sidecar.istio.io/includeOutboundIPRanges: "" # 出口トラフィックをインターセプトしない
proxy.istio.io/config: | # 環境変数 `OUTPUT_CERTS` を設定し、証明書を指定フォルダに出力
proxyMetadata:
OUTPUT_CERTS: /etc/istio-output-certs
sidecar.istio.io/userVolumeMount: '[{"name": "istio-certs", "mountPath": "/etc/istio-output-certs"}]' # Sidecar で共有ボリュームをマウント
{{< /text >}}

最後に、TLS メトリクス収集のために以下のように設定します：

{{< text yaml >}}
scheme: https
tls_config:
ca_file: /etc/prom-certs/root-cert.pem
cert_file: /etc/prom-certs/cert-chain.pem
key_file: /etc/prom-certs/key.pem
insecure_skip_verify: true # Prometheus は Istio のセキュアネームをサポートしないため、ターゲット Pod 証明書の検証をスキップします。
{{< /text >}}

## ベストプラクティス {#best-practices}

大規模メッシュでは、高度な設定で Prometheus をスケールさせることができます。詳細は[Prometheus で本番規模を監視](/ja/docs/ops/best-practices/observability/#using-prometheus-for-production-scale-monitoring)を参照してください。
