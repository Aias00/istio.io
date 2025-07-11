---
title: Istio メトリクスのカスタマイズ
description: このタスクでは、Istio メトリクスをカスタマイズする方法を紹介します。
weight: 25
keywords: [telemetry, metrics, customize]
owner: istio/wg-policies-and-telemetry-maintainers
test: yes
---

このタスクでは、Istio が生成するメトリクスをカスタマイズする方法を紹介します。

Istio はさまざまなダッシュボードで利用できるテレメトリデータを生成し、メッシュの情報を可視化できます。
例えば、Istio 対応ダッシュボードには以下があります：

- [Grafana](/zh/docs/tasks/observability/metrics/using-istio-dashboard/)
- [Kiali](/zh/docs/tasks/observability/kiali/)
- [Prometheus](/zh/docs/tasks/observability/metrics/querying-metrics/)

デフォルトでは、Istio は一連の標準メトリクス（例：`requests_total`）を定義・生成しますが、
[Telemetry API](/zh/docs/tasks/observability/telemetry/)
を使って標準メトリクスのカスタマイズや新規メトリクスの作成も可能です。

## 始める前に {#before-you-begin}

クラスタに[Istio をインストール](/zh/docs/setup/)し、アプリケーションをデプロイしてください。
または、Istio インストール時にカスタム統計を設定することもできます。

[Bookinfo サンプル](/zh/docs/examples/bookinfo/)アプリケーションはこのタスクの例として使われます。
インストール手順は [Bookinfo サンプルのデプロイ](/zh/docs/examples/bookinfo/#deploying-the-application) を参照してください。

## カスタムメトリクスの有効化 {#enable-custom-metrics}

例えば、以下のコマンドでテレメトリメトリクスをカスタマイズし、Gateway や Sidecar から発行される `requests_total` に
`request_host` と `destination_port` ディメンションを入出力両方向で追加できます：

{{< text bash >}}
$ cat <<EOF > ./custom_metrics.yaml
apiVersion: telemetry.istio.io/v1
kind: Telemetry
metadata:
name: namespace-metrics
spec:
metrics:

- providers: - name: prometheus
  overrides: - match:
  metric: REQUEST_COUNT
  tagOverrides:
  destination_port:
  value: "string(destination.port)"
  request_host:
  value: "request.host"
  EOF
  $ kubectl apply -f custom_metrics.yaml
  {{< /text >}}

## 結果の検証 {#verify-the-results}

メッシュにトラフィックを送信します。Bookinfo サンプルの場合、Web ブラウザで
`http://$GATEWAY_URL/productpage` にアクセスするか、次のコマンドを実行します：

{{< text bash >}}
$ curl "http://$GATEWAY_URL/productpage"
{{< /text >}}

{{< tip >}}
`$GATEWAY_URL` は [Bookinfo](/zh/docs/examples/bookinfo/) サンプルで設定した値です。
{{< /tip >}}

次のコマンドで、Istio が新しいディメンションや変更したディメンションのデータを生成しているか確認できます：

{{< text bash >}}
$ kubectl exec "$(kubectl get pod -l app=productpage -o jsonpath='{.items[0].metadata.name}')" -c istio-proxy -- curl -sS 'localhost:15000/stats/prometheus' | grep istio_requests_total
{{< /text >}}

例えば、出力の中で `istio_requests_total` 指標を探し、新しいディメンションが含まれているか確認してください。

{{< tip >}}
プロキシが設定を適用するまでに少し時間がかかる場合があります。指標が見つからない場合は、少し待ってから再度リクエストを送り、再度指標を探してください。
{{< /tip >}}

## 値に式を使う {#use-expressions-for-values}

メトリクス設定の値は一般的な式であり、JSON では文字列をダブルクォートで囲む必要があります（例："'string value'"）。
Mixer の式言語とは異なり、pipe（`|`）演算子はサポートされていませんが、
`has` や `in` 演算子を使って同様のことができます。例：

{{< text plain >}}
has(request.host) ? request.host : "unknown"
{{< /text >}}

詳細は[Common Expression Language（CEL）](https://opensource.google/projects/cel) を参照してください。

Istio ではすべての標準 [Envoy 属性](https://www.envoyproxy.io/docs/envoy/latest/intro/arch_overview/advanced/attributes) が利用可能です。
ピアメタデータは、出力属性 `upstream_peer` および入力属性 `downstream_peer` として利用でき、以下のフィールドを持ちます：

| フィールド  | 型       | 値                                         |
| ----------- | -------- | ------------------------------------------ |
| `app`       | `string` | アプリケーション名。                       |
| `version`   | `string` | アプリケーションのバージョン。             |
| `service`   | `string` | サービスインスタンス。                     |
| `revision`  | `string` | サービスのリビジョン。                     |
| `name`      | `string` | Pod 名。                                   |
| `namespace` | `string` | Pod のネームスペース。                     |
| `type`      | `string` | ワークロードタイプ。                       |
| `workload`  | `string` | ワークロード名。                           |
| `cluster`   | `string` | このワークロードが属するクラスタの識別子。 |

例えば、出力設定でピアの `app` ラベルを使う式は `filter_state.downstream_peer.app`
または `filter_state.upstream_peer.app` です。

## クリーンアップ {#cleanup}

`Bookinfo` サンプルアプリケーションとその設定を削除するには、[`Bookinfo` のクリーンアップ](/zh/docs/examples/bookinfo/#cleanup)を参照してください。
