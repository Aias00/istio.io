---
title: Envoy の統計情報
description: Envoy の統計情報を詳細に制御する方法。
weight: 10
aliases:
  - /zh/help/ops/telemetry/envoy-stats
  - /zh/docs/ops/telemetry/envoy-stats
owner: istio/wg-policies-and-telemetry-maintainers
test: yes
---

Envoy プロキシはネットワークトラフィックに関する詳細な統計情報を収集します。

Envoy の統計情報は特定の Envoy インスタンスのトラフィックのみをカバーします。[可観測性](/ja/docs/tasks/observability/)も参照し、サービスレベルの Istio テレメトリについて理解してください。Envoy プロキシが生成するこれらの統計データは、Pod インスタンスに関するより具体的な情報を提供します。

特定の Pod の統計情報を確認するには：

{{< text syntax=bash snip_id=get_stats >}}
$ kubectl exec "$POD" -c istio-proxy -- pilot-agent request GET stats
{{< /text >}}

Envoy は Pod の動作に関連する統計データを生成し、プロキシ機能ごとに統計範囲を限定します。
主な例は以下の通りです：

- [上流接続](https://www.envoyproxy.io/docs/envoy/latest/configuration/upstream/cluster_manager/cluster_stats)
- [リスナー](https://www.envoyproxy.io/docs/envoy/latest/configuration/listeners/stats)
- [HTTP コネクションマネージャ](https://www.envoyproxy.io/docs/envoy/latest/configuration/http/http_conn_man/stats)
- [TCP プロキシ](https://www.envoyproxy.io/docs/envoy/latest/configuration/listeners/network_filters/tcp_proxy_filter#statistics)
- [ルーター](https://www.envoyproxy.io/docs/envoy/latest/configuration/http/http_filters/router_filter.html?highlight=vhost#statistics)

Istio のデフォルト設定では、Envoy は最小限の統計情報のみを記録します。
これによりプロキシサーバー全体の CPU およびメモリ消費が抑えられます。デフォルトのキーワードセットは：

- `cluster_manager`
- `listener_manager`
- `server`
- `cluster.xds-grpc`

統計データ収集に関する Envoy の設定を確認するには、
[`istioctl proxy-config bootstrap`](/ja/docs/reference/commands/istioctl/#istioctl-proxy-config-bootstrap)
コマンドを使用できます。また、[Envoy 設定の詳細調査](/ja/docs/ops/diagnostic-tools/proxy-cmd/#deep-dive-into-envoy-configuration)も参照してください。
Envoy は `stats_matcher` JSON フィールドの `inclusion_list` に一致する統計データのみを収集します。

{{< tip >}}
注意：Envoy の統計データ名は、構成する Envoy の設定によって異なります。そのため、
Istio 管理下の Envoy の統計データ名は Istio の設定動作に影響されます。
Envoy ベースのダッシュボードやアラートを構築・運用している場合は、**Istio をアップグレードする前に**
[カナリア環境](/ja/docs/setup/upgrade/canary/index.md)で統計情報を必ず確認することを**強く推奨**します。
{{< /tip >}}

Istio プロキシでより多くの統計情報を記録したい場合は、
メッシュ設定に [`ProxyConfig.ProxyStatsMatcher`](/ja/docs/reference/config/istio.mesh.v1alpha1/#ProxyStatsMatcher) を追加できます。
たとえば、グローバルにサーキットブレーカー、リトライ、上流接続、リクエストタイムアウトの統計を有効にするには、
以下のような統計マッチ設定を指定します：

{{< tip >}}
統計マッチ設定を反映させるには、プロキシの再起動が必要です。
{{< /tip >}}

{{< text syntax=yaml snip_id=proxyStatsMatcher >}}
apiVersion: install.istio.io/v1alpha1
kind: IstioOperator
spec:
meshConfig:
defaultConfig:
proxyStatsMatcher:
inclusionRegexps: - "._outlier_detection._" - "._upstream_rq_retry._" - "._upstream*cx*._"
inclusionSuffixes: - "upstream_rq_timeout"
{{< /text >}}

各プロキシの `proxy.istio.io/config` アノテーションを使って、
グローバルな統計設定を上書きすることもできます。
たとえば、上記と同じ統計データを生成するには、Gateway プロキシやワークロードに次のアノテーションを追加します：

{{< text syntax=yaml snip_id=proxyIstioConfig >}}
metadata:
annotations:
proxy.istio.io/config: |-
proxyStatsMatcher:
inclusionRegexps: - "._outlier_detection._" - "._upstream_rq_retry._" - "._upstream*cx*._"
inclusionSuffixes: - "upstream_rq_timeout"
{{< /text >}}

{{< tip >}}
注意：`sidecar.istio.io/statsInclusionPrefixes`、
`sidecar.istio.io/statsInclusionRegexps`、`sidecar.istio.io/statsInclusionSuffixes` を使っている場合は、
`ProxyConfig` ベースの設定への移行を検討してください。これにより、Gateway と Sidecar プロキシの両方で
グローバルなデフォルトと一貫した上書き方法が提供されます。
{{< /tip >}}
