---
title: トレースサンプリングの設定
description: プロキシでトレースサンプリングを設定するさまざまな方法について学びます。
weight: 4
keywords: [sampling, telemetry, tracing, opentelemetry]
owner: istio/wg-policies-and-telemetry-maintainers
test: yes
---

Istio ではトレースサンプリングを設定する複数の方法が提供されています。
このページでは、サンプリングを設定するさまざまな方法について学びます。

## 始める前に {#before-you-begin}

1.  アプリケーションが[こちら](/zh/docs/tasks/observability/distributed-tracing/overview/)で説明されているようにトレースヘッダーを伝播していることを確認してください。

## 利用可能なトレースサンプリング設定 {#available-trace-sampling-configurations}

1.  パーセンテージサンプラー：トレース生成リクエストの割合をランダムにサンプリングします。

1.  カスタム OpenTelemetry サンプラー：`OpenTelemetryTracingProvider` と組み合わせて使用するカスタムサンプラー実装。

### パーセンテージサンプラー {#percentage-sampler}

{{< boilerplate telemetry-tracing-tips >}}

ランダムサンプリング率は、指定したパーセンテージ値を使ってサンプリングするリクエストを選択します。

サンプリング率は 0.0 から 100.0 の範囲で、精度は 0.01 です。
例えば、1 万リクエストごとに 5 件だけトレースしたい場合は、ここに 0.05 を指定します。

ランダムサンプリング率は 3 つの方法で設定できます：

#### Telemetry API {#telemetry-api}

さまざまなスコープ（メッシュ全体、名前空間、ワークロード）でサンプリングを設定でき、非常に柔軟です。
詳細は [Telemetry API](/zh/docs/tasks/observability/telemetry/) ドキュメントを参照してください。

Istio インストール時に `defaultConfig` で `sampling` を設定する必要はありません：

{{< text syntax=bash snip_id=install_without_sampling >}}
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

Telemetry API でトレースプロバイダーを有効化し、`randomSamplingPercentage` を設定します：

{{< text syntax=bash snip_id=enable_telemetry_with_sampling >}}
$ kubectl apply -f - <<EOF
apiVersion: telemetry.istio.io/v1
kind: Telemetry
metadata:
name: otel-demo
spec:
tracing:

- providers: - name: otel-tracing
  randomSamplingPercentage: 10
  EOF
  {{< /text >}}

#### `MeshConfig` の利用 {#using-meshconfig}

ランダムパーセンテージサンプリングは `MeshConfig` でグローバルに設定できます。

{{< text syntax=bash snip_id=install_default_sampling >}}
$ cat <<EOF | istioctl install -y -f -
apiVersion: install.istio.io/v1alpha1
kind: IstioOperator
spec:
meshConfig:
enableTracing: true
defaultConfig:
tracing:
sampling: 10
extensionProviders: - name: otel-tracing
opentelemetry:
port: 4317
service: opentelemetry-collector.observability.svc.cluster.local
resource_detectors:
environment: {}
EOF
{{< /text >}}

その後、Telemetry API でトレースプロバイダーを有効化します。
ここでは `randomSamplingPercentage` を設定しません。

{{< text syntax=bash snip_id=enable_telemetry_no_sampling >}}
$ kubectl apply -f - <<EOF
apiVersion: telemetry.istio.io/v1
kind: Telemetry
metadata:
name: mesh-default
namespace: istio-system
spec:
tracing:

- providers: - name: otel-tracing
  EOF
  {{< /text >}}

#### `proxy.istio.io/config` アノテーションの利用 {#using-the-proxy.istio.io/config-annotation}

Pod のメタデータ仕様に `proxy.istio.io/config` アノテーションを追加することで、
メッシュ全体のサンプリング設定を上書きできます。

例えば、上記のメッシュ全体のサンプリングを上書きするには、Pod マニフェストに以下を追加します：

{{< text syntax=yaml snip_id=none >}}
apiVersion: apps/v1
kind: Deployment
metadata:
name: curl
spec:
...
template:
metadata:
...
annotations:
...
proxy.istio.io/config: |
tracing:
sampling: 20
spec:
...
{{< /text >}}

### カスタム OpenTelemetry サンプラー {#custom-opentelemetry-sampler}

OpenTelemetry 仕様では [Sampler API](https://opentelemetry.io/docs/specs/otel/trace/sdk/#sampler) が定義されています。
Sampler API を使うことで、よりインテリジェントで効率的なサンプリング判断を行うカスタムサンプラーを構築できます。
例えば [確率サンプリング](https://opentelemetry.io/docs/specs/otel/trace/tracestate-probability-sampling-experimental/) などです。

このようなサンプラーは
[`OpenTelemetryTracingProvider`](/zh/docs/reference/config/istio.mesh.v1alpha1/#MeshConfig-ExtensionProvider-OpenTelemetryTracingProvider) と組み合わせて使用できます。

{{< quote >}}
プロキシ内に実装されたサンプラーは、
[Envoy OpenTelemetry Samplers](https://www.envoyproxy.io/docs/envoy/latest/api-v3/config/trace/opentelemetry/samplers#opentelemetry-samplers) で確認できます。
{{< /quote >}}

現在 Istio で利用できるカスタムサンプラー設定：

- [Dynatrace サンプラー](/zh/docs/reference/config/istio.mesh.v1alpha1/#MeshConfig-ExtensionProvider-OpenTelemetryTracingProvider-DynatraceSampler)

カスタムサンプラーは `MeshConfig` で設定します。以下は Dynatrace サンプラーの設定例です：

{{< text syntax=yaml snip_id=none >}}
apiVersion: install.istio.io/v1alpha1
kind: IstioOperator
spec:
meshConfig:
extensionProviders: - name: otel-tracing
opentelemetry:
port: 443
service: abc.live.dynatrace.com/api/v2/otlp
http:
path: "/api/v2/otlp/v1/traces"
timeout: 10s
headers: - name: "Authorization"
value: "Api-Token dt0c01."
dynatrace_sampler:
tenant: "abc"
cluster_id: 123
{{< /text >}}

### 優先順位 {#order-of-precedence}

複数の方法でサンプリングを設定できるため、それぞれの優先順位を理解することが重要です。

ランダムパーセンテージサンプラーを使う場合、優先順位は：

<table><tr><td>Telemetry API > Pod アノテーション > <code>MeshConfig</code> </td></tr></table>

つまり、上記すべてで値が定義されている場合、Telemetry API の値が選択されます。

カスタム OpenTelemetry サンプラーを設定する場合、優先順位は：

<table><tr><td>カスタム OTel サンプラー > （Telemetry API | Pod アノテーション | <code>MeshConfig</code>）</td></tr></table>

つまり、カスタム OpenTelemetry サンプラーが設定されている場合、それがすべての方法を上書きします。
また、ランダムパーセンテージ値は `100` に設定され、変更できません。これは、カスタムサンプラーが正しく判断するために 100% の Span を受け取る必要があるためです。

## OpenTelemetry Collector のデプロイ {#deploy-the0-opentelemetry-collector}

{{< boilerplate start-otel-collector-service >}}

## Bookinfo アプリケーションのデプロイ {#deploy-the-bookinfo-application}

[Bookinfo](/zh/docs/examples/bookinfo/#deploying-the-application) サンプルアプリケーションをデプロイします。

## Bookinfo サンプルでトレースを生成する {#generating-traces-using-the-bookinfo-sample}

1.  Bookinfo アプリケーションが起動し稼働している状態で、
    `http://$GATEWAY_URL/productpage` に一度または複数回アクセスしてトレース情報を生成します。

    {{< boilerplate trace-generation >}}

## クリーンアップ {#cleanup}

1.  Telemetry リソースを削除します：

    {{< text syntax=bash snip_id=cleanup_telemetry >}}
    $ kubectl delete telemetry otel-demo
    {{< /text >}}

1.  control-C を使うか、実行中の `istioctl` プロセスをすべて終了するか、以下のコマンドを実行してください：

    {{< text syntax=bash snip_id=none >}}
    $ istioctl uninstall --purge -y
    {{< /text >}}

1.  OpenTelemetry Collector をアンインストールします：

    {{< text syntax=bash snip_id=cleanup_collector >}}
    $ kubectl delete -f @samples/open-telemetry/otel.yaml@ -n observability
    $ kubectl delete namespace observability
    {{< /text >}}
