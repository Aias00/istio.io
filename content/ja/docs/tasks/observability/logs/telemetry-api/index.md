---
title: Telemetry API でアクセスログを設定する
description: このタスクでは、Telemetry API を使って Envoy プロキシがアクセスログを送信するように設定する方法を紹介します。
weight: 10
keywords: [telemetry, logs]
owner: istio/wg-policies-and-telemetry-maintainers
test: yes
---

Telemetry API は Istio のコア API としてしばらく前から導入されています。
以前はユーザーが Istio の `MeshConfig` で Telemetry を設定する必要がありました。

{{< boilerplate before-you-begin-egress >}}

{{< boilerplate start-httpbin-service >}}

## インストール {#installation}

この例では、[Grafana Loki](https://grafana.com/oss/loki/) にログを送信します。Loki がインストールされていることを確認してください。

{{< text syntax=bash snip_id=install_loki >}}
$ istioctl install -f @samples/open-telemetry/loki/iop.yaml@ --skip-confirmation
$ kubectl apply -f @samples/addons/loki.yaml@ -n istio-system
$ kubectl apply -f @samples/open-telemetry/loki/otel.yaml@ -n istio-system
{{< /text >}}

## Telemetry API の使い方 {#get-started-with-telemetry-api}

1. アクセスログ記録を有効化する

   {{< text bash >}}
   $ cat <<EOF | kubectl apply -n istio-system -f -
   apiVersion: telemetry.istio.io/v1
   kind: Telemetry
   metadata:
   name: mesh-logging-default
   spec:
   accessLogging:

   - providers: - name: otel
     EOF
     {{< /text >}}

   この例では組み込みの `envoy` アクセスログプロバイダーを使用しており、デフォルト設定以外は特に指定していません。

1. 特定のワークロードのアクセスログを無効化する

   次の設定で `curl` サービスのアクセスログを無効化できます：

   {{< text bash >}}
   $ cat <<EOF | kubectl apply -n default -f -
   apiVersion: telemetry.istio.io/v1
   kind: Telemetry
   metadata:
   name: disable-curl-logging
   namespace: default
   spec:
   selector:
   matchLabels:
   app: curl
   accessLogging:

   - providers: - name: otel
     disabled: true
     EOF
     {{< /text >}}

1. ワークロードモードでアクセスログをフィルタリングする

   次の設定で `httpbin` サービスのインバウンドアクセスログを無効化できます：

   {{< text bash >}}
   $ cat <<EOF | kubectl apply -n default -f -
   apiVersion: telemetry.istio.io/v1
   kind: Telemetry
   metadata:
   name: disable-httpbin-logging
   spec:
   selector:
   matchLabels:
   app: httpbin
   accessLogging:

   - providers: - name: otel
     match:
     mode: SERVER
     disabled: true
     EOF
     {{< /text >}}

1. CEL 式でアクセスログをフィルタリングする

   レスポンスコードが 500 以上の場合のみ、次の設定でアクセスログが出力されます：

   {{< text bash >}}
   $ cat <<EOF | kubectl apply -n default -f -
   apiVersion: telemetry.istio.io/v1alpha1
   kind: Telemetry
   metadata:
   name: filter-curl-logging
   spec:
   selector:
   matchLabels:
   app: curl
   accessLogging:

   - providers: - name: otel
     filter:
     expression: response.code >= 500
     EOF
     {{< /text >}}

   {{< tip >}}
   接続失敗時は `response.code` 属性がありません。
   この場合、CEL 式 `!has(response.code) || response.code >= 500` を使うべきです。
   {{< /tip >}}

1. CEL 式でデフォルトのアクセスログフィルタを設定する

   レスポンスコードが 400 以上、またはリクエストが BlackHoleCluster もしくは PassthroughCluster に送られた場合のみ、
   次の設定でアクセスログが出力されます（`xds.cluster_name` は Istio 1.16.2 以降で利用可能）：

   {{< text bash >}}
   $ cat <<EOF | kubectl apply -f -
   apiVersion: telemetry.istio.io/v1alpha1
   kind: Telemetry
   metadata:
   name: default-exception-logging
   namespace: istio-system
   spec:
   accessLogging:

   - providers:
     - name: otel
       filter:
       expression: "response.code >= 400 || xds.cluster_name == 'BlackHoleCluster' || xds.cluster_name == 'PassthroughCluster' "

   EOF
   {{< /text >}}

1. CEL 式でヘルスチェックアクセスログをフィルタリングする

   Amazon Route 53 のヘルスチェックサービスによるログでない場合のみ、次の設定でアクセスログが出力されます。
   注意：`request.useragent` は HTTP トラフィック専用なので、TCP トラフィックを壊さないように
   このフィールドの存在をチェックする必要があります。詳細は
   [CEL の型チェック](https://kubernetes.io/docs/reference/using-api/cel/#type-checking) を参照してください。

   {{< text bash >}}
   $ cat <<EOF | kubectl apply -f -
   apiVersion: telemetry.istio.io/v1alpha1
   kind: Telemetry
   metadata:
   name: filter-health-check-logging
   spec:
   accessLogging:

   - providers: - name: otel
     filter:
     expression: "!has(request.useragent) || !(request.useragent.startsWith(\"Amazon-Route53-Health-Check-Service\"))"
     EOF
     {{< /text >}}

   詳細は[値に式を使う](/zh/docs/tasks/observability/metrics/customize-metrics/#use-expressions-for-values)も参照してください。

## OpenTelemetry プロバイダーの利用 {#work-with-otel-provider}

Istio は [OpenTelemetry](https://opentelemetry.io/) プロトコルでのアクセスログ送信をサポートしています。
詳細は[こちら](/zh/docs/tasks/observability/logs/otel-provider)を参照してください。

## クリーンアップ {#cleanup}

1.  すべての Telemetry API を削除：

    {{< text bash >}}
    $ kubectl delete telemetry --all -A
    {{< /text >}}

1.  `loki` を削除：

    {{< text bash >}}
    $ kubectl delete -f @samples/addons/loki.yaml@ -n istio-system
    $ kubectl delete -f @samples/open-telemetry/loki/otel.yaml@ -n istio-system
    {{< /text >}}

1.  クラスタから Istio をアンインストール：

    {{< text bash >}}
    $ istioctl uninstall --purge --skip-confirmation
    {{< /text >}}
