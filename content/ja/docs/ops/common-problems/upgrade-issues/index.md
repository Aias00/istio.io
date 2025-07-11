---
title: アップグレードの問題
description: Istio のアップグレード時によく発生する問題の解決方法。
weight: 60
owner: istio/wg-policies-and-telemetry-maintainers
test: n/a
---

## EnvoyFilter の移行 {#envoyfilter-migration}

`EnvoyFilter` は、Istio の xDS 設定生成の実装詳細に密接に結びついた Alpha API です。
Istio のコントロールプレーンやデータプレーンをアップグレードする際は、`EnvoyFilter` Alpha API の使用に注意が必要です。
多くの場合、アップグレード時のリスクが低い標準の Istio API で `EnvoyFilter` を置き換えることができます。

### Telemetry API を使ったメトリクスのカスタマイズ {#use-telemetry-api-for-metrics- customization}

`IstioOperator` は、メトリクスフィルター設定を変更するためにテンプレート化された `EnvoyFilter` に依存していましたが、
Prometheus メトリクス生成のカスタマイズ方法は [Telemetry API](/ja/docs/tasks/observability/metrics/customize-metrics/) に置き換えられました。
この 2 つの方法は互換性がなく、Telemetry API は `EnvoyFilter` や `IstioOperator` のメトリクスカスタマイズ設定と併用できません。

例えば、以下の `IstioOperator` 設定は `destination_port` タグを追加します：

{{< text yaml >}}
apiVersion: install.istio.io/v1alpha1
kind: IstioOperator
spec:
values:
telemetry:
v2:
prometheus:
configOverride:
inboundSidecar:
metrics: - name: requests_total
dimensions:
destination_port: string(destination.port)
{{< /text >}}

次の `Telemetry` 設定は、上記の設定を置き換えます：

{{< text yaml >}}
apiVersion: telemetry.istio.io/v1
kind: Telemetry
metadata:
name: namespace-metrics
spec:
metrics:

- providers: - name: prometheus
  overrides: - match:
  metric: REQUEST_COUNT
  mode: SERVER
  tagOverrides:
  destination_port:
  value: "string(destination.port)"
  {{< /text >}}

### WasmPlugin API を使った Wasm データプレーンの拡張 {#use-wasmplugin-api-for-wasm-extensibility}

`EnvoyFilter` を使って Wasm フィルターを注入する方法は、
[WasmPlugin API](/ja/docs/tasks/extensibility/wasm-module-distribution) に置き換えられました。
WasmPlugin API では、イメージレジストリ、URL、またはローカルファイルからプラグインを動的にロードできます。
Wasm コードのデプロイにおいて「Null」プラグインランタイムは推奨されなくなりました。

### ゲートウェイトポロジーで信頼できるホップ数を設定 {#use-gateway-topology-to-set-the-number-of-trusted-hops}

HTTP コネクションマネージャーで信頼できるホップ数を設定するために `EnvoyFilter` を使う方法は、
[`ProxyConfig`](/ja/docs/ops/configuration/traffic-management/network-topologies) の [`gatewayTopology`](/ja/docs/reference/config/istio.mesh.v1alpha1/#Topology) フィールドに置き換えられました。
例えば、以下の `EnvoyFilter` 設定は、Pod またはゲートウェイでアノテーションを使って置き換えるのが推奨されます：

{{< text yaml >}}
apiVersion: networking.istio.io/v1alpha3
kind: EnvoyFilter
metadata:
name: ingressgateway-redirect-config
spec:
configPatches:

- applyTo: NETWORK_FILTER
  match:
  context: GATEWAY
  listener:
  filterChain:
  filter:
  name: envoy.filters.network.http_connection_manager
  patch:
  operation: MERGE
  value:
  typed_config:
  '@type': type.googleapis.com/envoy.extensions.filters.network.http_connection_manager.v3.HttpConnectionManager
  xff_num_trusted_hops: 1
  workloadSelector:
  labels:
  istio: ingress-gateway
  {{< /text >}}

同等のエントリーゲートウェイ Pod プロキシ設定アノテーション：

{{< text yaml >}}
metadata:
annotations:
"proxy.istio.io/config": '{"gatewayTopology" : { "numTrustedProxies": 1 }}'
{{< /text >}}

### ゲートウェイトポロジーでエントリーゲートウェイに PROXY プロトコルを有効化 {#use-gateway-topology-to-enable-proxy-protocol}

エントリーゲートウェイで [PROXY プロトコル](https://www.haproxy.org/download/1.8/doc/proxy-protocol.txt) を有効にするために `EnvoyFilter` を使う方法は、
[`ProxyConfig`](/ja/docs/ops/configuration/traffic-management/network-topologies) の [`gatewayTopology`](/ja/docs/reference/config/istio.mesh.v1alpha1/#Topology) フィールドに置き換えられました。
例えば、以下の `EnvoyFilter` 設定は、Pod またはメッシュでアノテーションを使って置き換えるのが推奨されます：

{{< text yaml >}}
apiVersion: networking.istio.io/v1alpha3
kind: EnvoyFilter
metadata:
name: proxy-protocol
spec:
configPatches:

- applyTo: LISTENER_FILTER
  patch:
  operation: INSERT_FIRST
  value:
  name: proxy_protocol
  typed_config:
  "@type": "type.googleapis.com/envoy.extensions.filters.listener.proxy_protocol.v3.ProxyProtocol"
  workloadSelector:
  labels:
  istio: ingress-gateway
  {{< /text >}}

同等のエントリーゲートウェイ Pod プロキシ設定アノテーション：

{{< text yaml >}}
metadata:
annotations:
"proxy.istio.io/config": '{"gatewayTopology" : { "proxyProtocol": {} }}'
{{< /text >}}

### プロキシアノテーションでヒストグラムバケットサイズをカスタマイズ {#use-proxy-annotation-to-customize-buckets}

`EnvoyFilter` と実験的なブートストラップディスカバリーサービスを使ってヒストグラムメトリクスのバケットサイズを設定する方法は、
プロキシアノテーション `sidecar.istio.io/statsHistogramBuckets` に置き換えられました。
例えば、以下の `EnvoyFilter` 設定は、Pod でアノテーションを使って置き換えるのが推奨されます：

{{< text yaml >}}
apiVersion: networking.istio.io/v1alpha3
kind: EnvoyFilter
metadata:
name: envoy-stats-1
namespace: istio-system
spec:
workloadSelector:
labels:
istio: ingressgateway
configPatches:

- applyTo: BOOTSTRAP
  patch:
  operation: MERGE
  value:
  stats_config:
  histogram_bucket_settings: - match:
  prefix: istiocustom
  buckets: [1,5,50,500,5000,10000]
  {{< /text >}}

同等の Pod アノテーション：

{{< text yaml >}}
metadata:
annotations:
"sidecar.istio.io/statsHistogramBuckets": '{"istiocustom":[1,5,50,500,5000,10000]}'
{{< /text >}}
