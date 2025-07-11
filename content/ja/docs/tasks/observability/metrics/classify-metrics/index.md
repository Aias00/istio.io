---
title: リクエストやレスポンスに基づくメトリクスの分類
description: このタスクでは、リクエストやレスポンスのタイプごとにグループ化してテレメトリを改善する方法を紹介します。
weight: 27
keywords: [telemetry, metrics, classify, request-based, openapispec, swagger]
owner: istio/wg-policies-and-telemetry-maintainers
test: no
---

メッシュ内のサービスが処理するリクエストやレスポンスのタイプごとにテレメトリデータを可視化することは非常に有用です。
例えば、書店が書評リクエストの回数を追跡したい場合、書評リクエストは次のような構造になります：

{{< text plain >}}
GET /reviews/{review_id}
{{< /text >}}

書評リクエストの回数を数えるには、無限に変化する `review_id` を考慮する必要があります。
`GET /reviews/1` の直後に `GET /reviews/2` が来ても、どちらも書評取得リクエストとしてカウントすべきです。

Istio では AttributeGen プラグインを使って分類ルールを作成でき、このプラグインでリクエストを固定数の論理操作にグループ化できます。
例えば、[`Open API Spec operationId`](https://swagger.io/docs/specification/paths-and-operations/) を使って `GetReviews` という操作名を作成できます。
この情報は `istio_operationId` 属性としてリクエスト処理に注入され、値は `GetReviews` となります。
この属性は Istio 標準メトリクスのディメンションとして利用できます。
同様に、`ListReviews` や `CreateReviews` など他の操作ごとにメトリクスを追跡できます。

## リクエストごとにメトリクスを分類する {#classify-metrics-by-request}

リクエストのタイプごと（例：`ListReview`、`GetReview`、`CreateReview`）に分類できます。

1.  例えば `attribute_gen_service.yaml` というファイルを作成し、以下の内容で保存します。
    これにより `istio.attributegen` プラグインが追加され、`istio_operationId` 属性が作成され、分類値でこの属性が埋められます。

        この設定はサービス固有です。なぜならリクエストパスは通常サービスごとに異なるためです。

        {{< text yaml >}}

    apiVersion: extensions.istio.io/v1alpha1
    kind: WasmPlugin
    metadata:
    name: istio-attributegen-filter
    spec:
    selector:
    matchLabels:
    app: reviews
    url: https://storage.googleapis.com/istio-build/proxy/attributegen-359dcd3a19f109c50e97517fe6b1e2676e870c4d.wasm
    imagePullPolicy: Always
    phase: AUTHN
    pluginConfig:
    attributes: - output_attribute: "istio_operationId"
    match: - value: "ListReviews"
    condition: "request.url_path == '/reviews' && request.method == 'GET'" - value: "GetReview"
    condition: "request.url_path.matches('^/reviews/[[:alnum:]]\*$') && request.method == 'GET'" - value: "CreateReview"
    condition: "request.url_path == '/reviews/' && request.method == 'POST'"

---

apiVersion: telemetry.istio.io/v1
kind: Telemetry
metadata:
name: custom-tags
spec:
metrics: - overrides: - match:
metric: REQUEST_COUNT
mode: CLIENT_AND_SERVER
tagOverrides:
request_operation:
value: filter_state['wasm.istio_operationId']
providers: - name: prometheus
{{< /text >}}

1. 次のコマンドで設定を適用します：

   {{< text bash >}}
   $ kubectl -n istio-system apply -f attribute_gen_service.yaml
   {{< /text >}}

1. 変更が反映されたら、Prometheus で `reviews` Pod の `istio_requests_total` など新しい/変更されたディメンションを確認します。

## レスポンスごとにメトリクスを分類する {#classify-metrics-by-response}

リクエストと同様の手順でレスポンスごとに分類できます。なお、`response_code` はデフォルトでディメンションとして存在しますが、下記例ではその値の付け方を変更しています。

1.  例えば `attribute_gen_service.yaml` というファイルを作成し、以下の内容で保存します。
    これにより `istio.attributegen` プラグインが追加され、統計プラグイン用の `istio_responseClass` 属性が生成されます。

        この例では、さまざまなレスポンスを分類し、例えば 200 番台のコードをすべて `2xx` ディメンションにまとめます。

        {{< text yaml >}}

    apiVersion: extensions.istio.io/v1alpha1
    kind: WasmPlugin
    metadata:
    name: istio-attributegen-filter
    spec:
    selector:
    matchLabels:
    app: productpage
    url: https://storage.googleapis.com/istio-build/proxy/attributegen-359dcd3a19f109c50e97517fe6b1e2676e870c4d.wasm
    imagePullPolicy: Always
    phase: AUTHN
    pluginConfig:
    attributes: - output_attribute: istio_responseClass
    match: - value: 2xx
    condition: response.code >= 200 && response.code <= 299 - value: 3xx
    condition: response.code >= 300 && response.code <= 399 - value: "404"
    condition: response.code == 404 - value: "429"
    condition: response.code == 429 - value: "503"
    condition: response.code == 503 - value: 5xx
    condition: response.code >= 500 && response.code <= 599 - value: 4xx
    condition: response.code >= 400 && response.code <= 499

---

apiVersion: telemetry.istio.io/v1
kind: Telemetry
metadata:
name: custom-tags
spec:
metrics: - overrides: - match:
metric: REQUEST_COUNT
mode: CLIENT_AND_SERVER
tagOverrides:
response_code:
value: filter_state['wasm.istio_responseClass']
providers: - name: prometheus
{{< /text >}}

1. 次のコマンドで設定を適用します：

   {{< text bash >}}
   $ kubectl -n istio-system apply -f attribute_gen_service.yaml
   {{< /text >}}

## 結果の検証 {#verify-the-results}

1. アプリケーションにトラフィックを送信してメトリクスを生成します。

1. Prometheus で `2xx` など新しい/変更されたディメンションを確認します。
   または、次のコマンドで Istio が新しいディメンションのデータを生成しているか確認できます：

   {{< text bash >}}
   $ kubectl exec pod-name -c istio-proxy -- curl -sS 'localhost:15000/stats/prometheus' | grep istio\_
   {{< /text >}}

   出力の中で、（例：`istio_requests_total` などの）メトリクスに新しい/変更されたディメンションが存在するか確認してください。

## トラブルシューティング {#troubleshooting}

分類が期待通りに動作しない場合は、以下の原因や対処法を確認してください。

設定変更を適用した Service に対応する Pod の Envoy プロキシログを確認します。
次のコマンドで分類を設定した Pod（`pod-name`）の Envoy プロキシログにサービスエラーがないか確認してください：

{{< text bash >}}
$ kubectl logs pod-name -c istio-proxy | grep -e "Config Error" -e "envoy wasm"
{{< /text >}}

また、次のコマンドの出力で再起動の兆候がないか確認し、Envoy プロキシがクラッシュしていないことを確認してください：

{{< text bash >}}
$ kubectl get pods pod-name
{{< /text >}}

## クリーンアップ {#cleanup}

yaml 設定ファイルを削除します。

{{< text bash >}}
$ kubectl -n istio-system delete -f attribute_gen_service.yaml
{{< /text >}}
