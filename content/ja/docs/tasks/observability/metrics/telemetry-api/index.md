---
title: Telemetry API で Istio メトリクスをカスタマイズする
description: このタスクでは、Telemetry API を使って Istio メトリクスをカスタマイズする方法を紹介します。
weight: 10
keywords: [telemetry, metrics, customize]
owner: istio/wg-policies-and-telemetry-maintainers
test: no
---

Telemetry API は現在、Istio の主流 API となっています。
以前は、ユーザーは Istio の監視設定セクション `telemetry` でメトリクスを設定する必要がありました。

このタスクでは、Telemetry API を使って Istio のテレメトリメトリクスをカスタマイズする方法を紹介します。

## 始める前に{#before-you-begin}

クラスタに[Istio をインストール](/zh/docs/setup/)し、アプリケーションをデプロイしてください。

注意：Telemetry API は `EnvoyFilter` と併用できません。
詳細は [issue](https://github.com/istio/istio/issues/39772) を参照してください。

- Istio バージョン `1.18` 以降では、Prometheus の `EnvoyFilter` はデフォルトでインストールされず、
  代わりに `meshConfig.defaultProviders` で有効化されます。テレメトリのカスタマイズには Telemetry API を使用してください。

- Istio `1.18` より前のバージョンでは、以下の `IstioOperator` 設定でインストールしてください：

{{< text yaml >}}
apiVersion: install.istio.io/v1alpha1
kind: IstioOperator
spec:
values:
telemetry:
enabled: true
v2:
enabled: false
{{< /text >}}

## メトリクスの上書き{#override-metrics}

`metrics` セクションでは、メトリクスのディメンション値の式を指定したり、既存ディメンションの削除や上書きができます。
`tags_to_remove` を使うか、ディメンションを再定義することで標準メトリクスの定義を変更できます。

1. `REQUEST_COUNT` メトリクスから `grpc_response_status` タグを削除する

   {{< text yaml >}}
   apiVersion: telemetry.istio.io/v1
   kind: Telemetry
   metadata:
   name: remove-tags
   namespace: istio-system
   spec:
   metrics: - providers: - name: prometheus
   overrides: - match:
   mode: CLIENT_AND_SERVER
   metric: REQUEST_COUNT
   tagOverrides:
   grpc_response_status:
   operation: REMOVE
   {{< /text >}}

1. `REQUEST_COUNT` メトリクスにカスタムタグを追加する

   {{< text yaml >}}
   apiVersion: telemetry.istio.io/v1
   kind: Telemetry
   metadata:
   name: custom-tags
   namespace: istio-system
   spec:
   metrics: - overrides: - match:
   metric: REQUEST_COUNT
   mode: CLIENT
   tagOverrides:
   destination_x:
   value: filter_state.upstream_peer.app - match:
   metric: REQUEST_COUNT
   mode: SERVER
   tagOverrides:
   source_x:
   value: filter_state.downstream_peer.app
   providers: - name: prometheus
   {{< /text >}}

## メトリクスの無効化{#disable-metrics}

1. 以下の設定で全メトリクスを無効化：

   {{< text yaml >}}
   apiVersion: telemetry.istio.io/v1
   kind: Telemetry
   metadata:
   name: remove-all-metrics
   namespace: istio-system
   spec:
   metrics: - providers: - name: prometheus
   overrides: - disabled: true
   match:
   mode: CLIENT_AND_SERVER
   metric: ALL_METRICS
   {{< /text >}}

1. 以下の設定で `REQUEST_COUNT` メトリクスを無効化：

   {{< text yaml >}}
   apiVersion: telemetry.istio.io/v1
   kind: Telemetry
   metadata:
   name: remove-request-count
   namespace: istio-system
   spec:
   metrics: - providers: - name: prometheus
   overrides: - disabled: true
   match:
   mode: CLIENT_AND_SERVER
   metric: REQUEST_COUNT
   {{< /text >}}

1. 以下の設定でクライアント側の `REQUEST_COUNT` メトリクスを無効化：

   {{< text yaml >}}
   apiVersion: telemetry.istio.io/v1
   kind: Telemetry
   metadata:
   name: remove-client
   namespace: istio-system
   spec:
   metrics: - providers: - name: prometheus
   overrides: - disabled: true
   match:
   mode: CLIENT
   metric: REQUEST_COUNT
   {{< /text >}}

1. 以下の設定でサーバ側の `REQUEST_COUNT` メトリクスを無効化：

   {{< text yaml >}}
   apiVersion: telemetry.istio.io/v1
   kind: Telemetry
   metadata:
   name: remove-server
   namespace: istio-system
   spec:
   metrics: - providers: - name: prometheus
   overrides: - disabled: true
   match:
   mode: SERVER
   metric: REQUEST_COUNT
   {{< /text >}}

## 結果の検証 {#verify-the-results}

メッシュにトラフィックを送信します。Bookinfo サンプルの場合、Web ブラウザで `http://$GATEWAY_URL/productpage` にアクセスするか、次のコマンドを実行します：

{{< text bash >}}
$ curl "http://$GATEWAY_URL/productpage"
{{< /text >}}

{{< tip >}}
`$GATEWAY_URL` の値は [Bookinfo](/zh/docs/examples/bookinfo/) サンプルで設定されています。
{{< /tip >}}

次のコマンドで、Istio が新しいディメンションや変更したディメンションのデータを生成しているか確認できます：

{{< text bash >}}
$ istioctl x es "$(kubectl get pod -l app=productpage -o jsonpath='{.items[0].metadata.name}')" -oprom | grep istio_requests_total | grep -v TYPE |grep -v 'reporter="destination"'
{{< /text >}}

{{< text bash >}}
$ istioctl x es "$(kubectl get pod -l app=details -o jsonpath='{.items[0].metadata.name}')" -oprom | grep istio_requests_total
{{< /text >}}

例えば、出力の中で `istio_requests_total` 指標を探し、新しいディメンションが含まれているか確認してください。

{{< tip >}}
プロキシが設定を適用するまでに少し時間がかかる場合があります。メトリクスが見つからない場合は、少し待ってから再度リクエストを送り、再度メトリクスを探してください。
{{< /tip >}}
