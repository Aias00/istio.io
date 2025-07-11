---
title: OpenTelemetry
description: プロキシを設定して OpenTelemetry 形式でトレースを送信する方法を学びます。
weight: 5
keywords: [telemetry, tracing, opentelemetry, span, port-forwarding]
aliases:
  - /zh/docs/tasks/telemetry/distributed-tracing/opentelemetry/
  - /zh/docs/tasks/observability/distributed-tracing/lightstep/
  - /zh/latest/docs/tasks/observability/distributed-tracing/lightstep/
owner: istio/wg-policies-and-telemetry-maintainers
test: yes
---

[OpenTelemetry](https://opentelemetry.io/) (OTel) は、ベンダーに依存しないオープンソースのオブザーバビリティフレームワークであり、
テレメトリデータの検出、生成、収集、エクスポートを行います。
[OpenTelemetry プロトコル](https://opentelemetry.io/docs/specs/otlp/) (OTLP) トレースは、
[Jaeger](/zh/docs/tasks/observability/distributed-tracing/jaeger/) や多くの商用サービスに送信できます。

Istio がどのようにトレースを処理するかについては、このタスクの[概要](../overview/)をご覧ください。

このタスクを完了すると、どの言語、フレームワーク、またはプラットフォームでアプリケーションを構築していても、
[OpenTelemetry](https://www.opentelemetry.io/) のトレースにアプリケーションを参加させる方法が分かります。

このタスクでは [Bookinfo](/zh/docs/examples/bookinfo/) サンプルアプリケーションを使用し、
[OpenTelemetry Collector](https://opentelemetry.io/docs/collector/) をトレースレシーバーとして利用します。
OTLP 互換バックエンドに直接トレースを送信する例については、
[Jaeger タスク](/zh/docs/tasks/observability/distributed-tracing/jaeger/) を参照してください。

Istio がどのようにトレースを処理するかについては、このタスクの[概要](../overview/)をご覧ください。

## OpenTelemetry Collector のデプロイ {#deploy-the-opentelemetry-collector}

{{< boilerplate start-otel-collector-service >}}

## インストール {#installation}

すべてのトレースオプションは `MeshConfig` でグローバルに設定できます。
設定を簡単にするため、`istioctl install -f` コマンドに渡せる YAML ファイルを作成することを推奨します。

## Exporter の選択 {#choosing-the-exporter}

Istio は gRPC または HTTP を使って
[OpenTelemetry Protocol（OTLP）](https://opentelemetry.io/docs/specs/otel/protocol/) トレースをエクスポートできます。
一度に設定できる Exporter は 1 つ（gRPC または HTTP のいずれか）です。

### gRPC でエクスポート {#exporting-via-grpc}

この例では、トレースは OTLP/gRPC で OpenTelemetry Collector にエクスポートされます。
また、[環境リソースディテクター](https://opentelemetry.io/docs/languages/js/resources/#adding-resources-with-environment-variables)も有効になっています。
環境ディテクターは、環境変数 `OTEL_RESOURCE_ATTRIBUTES` の属性をエクスポートされる OpenTelemetry リソースに追加します。

{{< text syntax=bash snip_id=none >}}
$ cat <<EOF | istioctl install -y -f -
apiVersion: install.istio.io/v1alpha1
kind: IstioOperator
spec:
meshConfig:
enableTracing: true
extensionProviders: - name: otel-tracing
opentelemetry:
port: 4317
service: opentelemetry-collector.observability.svc.cluster.local
resource_detectors:
environment: {}
EOF
{{< /text >}}

### HTTP でエクスポート {#exporting-via-http}

この例では、トレースは OTLP/HTTP で OpenTelemetry Collector にエクスポートされます。
また、[環境リソースディテクター](https://opentelemetry.io/docs/languages/js/resources/#adding-resources-with-environment-variables)も有効になっています。
環境ディテクターは、環境変数 `OTEL_RESOURCE_ATTRIBUTES` の属性をエクスポートされる OpenTelemetry リソースに追加します。

{{< text syntax=bash snip_id=install_otlp_http >}}
$ cat <<EOF | istioctl install -y -f -
apiVersion: install.istio.io/v1alpha1
kind: IstioOperator
spec:
meshConfig:
enableTracing: true
extensionProviders: - name: otel-tracing
opentelemetry:
port: 4318
service: opentelemetry-collector.observability.svc.cluster.local
http:
path: "/v1/traces"
timeout: 5s
headers: - name: "custom-header"
value: "custom value"
resource_detectors:
environment: {}
EOF
{{< /text >}}

## Telemetry API でメッシュのトレースを有効化 {#enable-tracing-for-mesh-via-telemetry-api}

以下の設定を適用してトレースを有効にします：

{{< text syntax=bash snip_id=enable_telemetry >}}
$ kubectl apply -f - <<EOF
apiVersion: telemetry.istio.io/v1
kind: Telemetry
metadata:
name: otel-demo
spec:
tracing:

- providers: - name: otel-tracing
  randomSamplingPercentage: 100
  customTags:
  "my-attribute":
  literal:
  value: "default-value"
  EOF
  {{< /text >}}

## Bookinfo アプリケーションのデプロイ {#deploy-the-bookinfo-application}

[Bookinfo](/zh/docs/examples/bookinfo/#deploying-the-application)
サンプルアプリケーションをデプロイします。

## Bookinfo サンプルでトレースを生成する {#generating-traces-using-the-bookinfo-sample}

1.  Bookinfo アプリケーションが起動し稼働している状態で、
    `http://$GATEWAY_URL/productpage` に一度または複数回アクセスしてトレース情報を生成します。

    {{< boilerplate trace-generation >}}

1.  サンプルで使用されている OpenTelemetry Collector は、トレースをコンソールにエクスポートするように設定されています。
    サンプルの Collector 設定を使用している場合、Collector のログを確認することでトレースが到達しているか検証できます。ログには次のような内容が含まれるはずです：

    {{< text syntax=yaml snip_id=none >}}
    Resource SchemaURL:
    Resource labels:
    -> service.name: STRING(productpage.default)
    ScopeSpans #0
    ScopeSpans SchemaURL:
    InstrumentationScope
    Span #0
    Trace ID : 79fb7b59c1c3a518750a5d6dad7cd2d1
    Parent ID : 0cf792b061f0ad51
    ID : 2dff26f3b4d6d20f
    Name : egress reviews:9080
    Kind : SPAN_KIND_CLIENT
    Start time : 2024-01-30 15:57:58.588041 +0000 UTC
    End time : 2024-01-30 15:57:59.451116 +0000 UTC
    Status code : STATUS_CODE_UNSET
    Status message :
    Attributes:
    -> node_id: STRING(sidecar~10.244.0.8~productpage-v1-564d4686f-t6s4m.default~default.svc.cluster.local)
    -> zone: STRING()
    -> guid:x-request-id: STRING(da543297-0dd6-998b-bd29-fdb184134c8c)
    -> http.url: STRING(http://reviews:9080/reviews/0)
    -> http.method: STRING(GET)
    -> downstream_cluster: STRING(-)
    -> user_agent: STRING(curl/7.74.0)
    -> http.protocol: STRING(HTTP/1.1)
    -> peer.address: STRING(10.244.0.8)
    -> request_size: STRING(0)
    -> response_size: STRING(441)
    -> component: STRING(proxy)
    -> upstream_cluster: STRING(outbound|9080||reviews.default.svc.cluster.local)
    -> upstream_cluster.name: STRING(outbound|9080||reviews.default.svc.cluster.local)
    -> http.status_code: STRING(200)
    -> response_flags: STRING(-)
    -> istio.namespace: STRING(default)
    -> istio.canonical_service: STRING(productpage)
    -> istio.mesh_id: STRING(cluster.local)
    -> istio.canonical_revision: STRING(v1)
    -> istio.cluster_id: STRING(Kubernetes)
    -> my-attribute: STRING(default-value)
    {{< /text >}}

## クリーンアップ {#cleanup}

1.  Telemetry リソースを削除します：

    {{< text syntax=bash snip_id=cleanup_telemetry >}}
    $ kubectl delete telemetry otel-demo
    {{< /text >}}

1.  Ctrl+C を使うか、実行中の `istioctl` プロセスをすべて終了するか、以下のコマンドを実行してください：

    {{< text syntax=bash snip_id=none >}}
    $ killall istioctl
    {{< /text >}}

1.  OpenTelemetry Collector をアンインストールします：

    {{< text syntax=bash snip_id=cleanup_collector >}}
    $ kubectl delete -f @samples/open-telemetry/otel.yaml@ -n observability
    $ kubectl delete namespace observability
    {{< /text >}}

1.  今後のタスクを試す予定がなければ、
    [Bookinfo のクリーンアップ](/zh/docs/examples/bookinfo/#cleanup)の手順に従ってアプリケーション全体を停止してください。
