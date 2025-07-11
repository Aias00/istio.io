---
title: OpenTelemetry
description: このタスクでは、OpenTelemetry コレクターを使って Envoy プロキシがアクセスログを送信するように設定する方法を紹介します。
weight: 10
keywords: [telemetry, logs]
owner: istio/wg-policies-and-telemetry-maintainers
test: yes
---

Envoy プロキシを
[OpenTelemetry フォーマット](https://www.envoyproxy.io/docs/envoy/latest/api-v3/extensions/access_loggers/open_telemetry/v3/logs_service.proto)
で[アクセスログ](https://www.envoyproxy.io/docs/envoy/latest/configuration/observability/access_log/usage)をエクスポートするように設定できます。
この例では、これらのプロキシはログを標準出力に出力するよう設定された
[OpenTelemetry コレクター](https://github.com/open-telemetry/opentelemetry-collector)にアクセスログを送信します。
その後、`kubectl logs` コマンドで OpenTelemetry コレクターの標準出力を確認できます。

{{< boilerplate before-you-begin-egress >}}

{{< boilerplate start-httpbin-service >}}

{{< boilerplate start-otel-collector-service >}}

## Envoy のアクセスログ記録を有効化する {#enable-envoy-access-logging}

アクセスログ記録を有効化するには、[Telemetry API](/zh/docs/tasks/observability/telemetry/) を利用できます。

`MeshConfig` を編集し、`otel` という名前の OpenTelemetry プロバイダーを追加します。
以下の拡張プロバイダーのスニペットを追加します：

{{< text yaml >}}
extensionProviders:

- name: otel
  envoyOtelAls:
  service: opentelemetry-collector.observability.svc.cluster.local
  port: 4317
  {{< /text >}}

最終的な設定は次のようになります：

{{< text yaml >}}
apiVersion: v1
kind: ConfigMap
metadata:
name: istio
namespace: istio-system
data:
mesh: |-
accessLogFile: /dev/stdout
defaultConfig:
discoveryAddress: istiod.istio-system.svc:15012
proxyMetadata: {}
tracing:
zipkin:
address: zipkin.istio-system:9411
enablePrometheusMerge: true
extensionProviders: - name: otel
envoyOtelAls:
service: opentelemetry-collector.observability.svc.cluster.local
port: 4317
rootNamespace: istio-system
trustDomain: cluster.local
meshNetworks: 'networks: {}'
{{< /text >}}

次に、Telemetry リソースを追加して Istio にアクセスログを OpenTelemetry コレクターへ送信するよう指示します。

{{< text bash >}}
$ cat <<EOF | kubectl apply -n default -f -
apiVersion: telemetry.istio.io/v1
kind: Telemetry
metadata:
name: curl-logging
spec:
selector:
matchLabels:
app: curl
accessLogging: - providers: - name: otel
EOF
{{< /text >}}

上記の例では `otel` アクセスログプロバイダーを使用しており、デフォルト設定以外は特に指定していません。

同様の設定は個別の名前空間やワークロード単位にも適用でき、きめ細かいログ制御が可能です。

Telemetry API の詳細は [Telemetry API 概要](/zh/docs/tasks/observability/telemetry/) をご覧ください。

### メッシュ設定を使う {#using-mesh-config}

`IstioOperator` 設定で Istio をインストールしている場合は、次のフィールドを設定に追加してください：

{{< text yaml >}}
spec:
meshConfig:
accessLogFile: /dev/stdout
extensionProviders: - name: otel
envoyOtelAls:
service: opentelemetry-collector.observability.svc.cluster.local
port: 4317
defaultProviders:
accessLogging: - envoy - otel
{{< /text >}}

または、元の `istioctl install` コマンドに同等の設定を追加します。例：

{{< text syntax=bash snip_id=none >}}
$ istioctl install -f <your-istio-operator-config-file>
{{< /text >}}

## デフォルトのアクセスログフォーマット {#default-access-log-format}

`accessLogFormat` が指定されていない場合、Istio は次のデフォルトのアクセスログフォーマットを使用します：

{{< text plain >}}
[%START_TIME%] "%REQ(:METHOD)% %REQ(X-ENVOY-ORIGINAL-PATH?:PATH)% %PROTOCOL%" %RESPONSE_CODE% %RESPONSE_FLAGS% %RESPONSE_CODE_DETAILS% %CONNECTION_TERMINATION_DETAILS%
"%UPSTREAM_TRANSPORT_FAILURE_REASON%" %BYTES_RECEIVED% %BYTES_SENT% %DURATION% %RESP(X-ENVOY-UPSTREAM-SERVICE-TIME)% "%REQ(X-FORWARDED-FOR)%" "%REQ(USER-AGENT)%" "%REQ(X-REQUEST-ID)%"
"%REQ(:AUTHORITY)%" "%UPSTREAM_HOST%" %UPSTREAM_CLUSTER% %UPSTREAM_LOCAL_ADDRESS% %DOWNSTREAM_LOCAL_ADDRESS% %DOWNSTREAM_REMOTE_ADDRESS% %REQUESTED_SERVER_NAME% %ROUTE_NAME%\n
{{< /text >}}

下表はデフォルトのアクセスログフォーマットを使った例で、`curl` から `httpbin` へのリクエスト時の出力です：

| ログ演算子                                                       | curl のアクセスログ                                                      | httpbin のアクセスログ                            |
| ---------------------------------------------------------------- | ------------------------------------------------------------------------ | ------------------------------------------------- |
| `[%START_TIME%]`                                                 | `[2020-11-25T21:26:18.409Z]`                                             | `[2020-11-25T21:26:18.409Z]`                      |
| `"%REQ(:METHOD)% %REQ(X-ENVOY-ORIGINAL-PATH?:PATH)% %PROTOCOL%"` | `"GET /status/418 HTTP/1.1"`                                             | `"GET /status/418 HTTP/1.1"`                      |
| `%RESPONSE_CODE%`                                                | `418`                                                                    | `418`                                             |
| `%RESPONSE_FLAGS%`                                               | `-`                                                                      | `-`                                               |
| `%RESPONSE_CODE_DETAILS%`                                        | `via_upstream`                                                           | `via_upstream`                                    |
| `%CONNECTION_TERMINATION_DETAILS%`                               | `-`                                                                      | `-`                                               |
| `"%UPSTREAM_TRANSPORT_FAILURE_REASON%"`                          | `"-"`                                                                    | `"-"`                                             |
| `%BYTES_RECEIVED%`                                               | `0`                                                                      | `0`                                               |
| `%BYTES_SENT%`                                                   | `135`                                                                    | `135`                                             |
| `%DURATION%`                                                     | `4`                                                                      | `3`                                               |
| `%RESP(X-ENVOY-UPSTREAM-SERVICE-TIME)%`                          | `4`                                                                      | `1`                                               |
| `"%REQ(X-FORWARDED-FOR)%"`                                       | `"-"`                                                                    | `"-"`                                             |
| `"%REQ(USER-AGENT)%"`                                            | `"curl/7.73.0-DEV"`                                                      | `"curl/7.73.0-DEV"`                               |
| `"%REQ(X-REQUEST-ID)%"`                                          | `"84961386-6d84-929d-98bd-c5aee93b5c88"`                                 | `"84961386-6d84-929d-98bd-c5aee93b5c88"`          |
| `"%REQ(:AUTHORITY)%"`                                            | `"httpbin:8000"`                                                         | `"httpbin:8000"`                                  |
| `"%UPSTREAM_HOST%"`                                              | `"10.44.1.27:80"`                                                        | `"127.0.0.1:80"`                                  |
| `%UPSTREAM_CLUSTER%`                                             | <code>outbound&#124;8000&#124;&#124;httpbin.foo.svc.cluster.local</code> | <code>inbound&#124;8000&#124;&#124;</code>        |
| `%UPSTREAM_LOCAL_ADDRESS%`                                       | `10.44.1.23:37652`                                                       | `127.0.0.1:41854`                                 |
| `%DOWNSTREAM_LOCAL_ADDRESS%`                                     | `10.0.45.184:8000`                                                       | `10.44.1.27:80`                                   |
| `%DOWNSTREAM_REMOTE_ADDRESS%`                                    | `10.44.1.23:46520`                                                       | `10.44.1.23:37652`                                |
| `%REQUESTED_SERVER_NAME%`                                        | `-`                                                                      | `outbound_.8000_._.httpbin.foo.svc.cluster.local` |
| `%ROUTE_NAME%`                                                   | `default`                                                                | `default`                                         |

## アクセスログのテスト {#test-access-log}

1.  `curl` から `httpbin` へリクエストを送信します：

    {{< text bash >}}
    $ kubectl exec "$SOURCE_POD" -c curl -- curl -sS -v httpbin:8000/status/418
    ...
    < HTTP/1.1 418 Unknown
    ...
    < server: envoy
    ...
    I'm a teapot!
    ...
    {{< /text >}}

1.  `otel-collector` のログを確認します：

    {{< text bash >}}
    $ kubectl logs -l app=opentelemetry-collector -n observability
    [2020-11-25T21:26:18.409Z] "GET /status/418 HTTP/1.1" 418 - via*upstream - "-" 0 135 3 1 "-" "curl/7.73.0-DEV" "84961386-6d84-929d-98bd-c5aee93b5c88" "httpbin:8000" "127.0.0.1:80" inbound|8000|| 127.0.0.1:41854 10.44.1.27:80 10.44.1.23:37652 outbound*.8000*.*.httpbin.foo.svc.cluster.local default
    {{< /text >}}

リクエストに対応するメッセージが送信元と宛先（`curl` と `httpbin`）の Istio プロキシログにそれぞれ出力されていることに注目してください。
このログには HTTP メソッド（`GET`）、HTTP パス（`/status/418`）、レスポンスコード（`418`）やその他の[リクエスト関連情報](https://www.envoyproxy.io/docs/envoy/latest/configuration/observability/access_log/usage#format-rules)が含まれています。

## クリーンアップ {#cleanup}

[curl]({{< github_tree >}}/samples/curl) と [httpbin]({{< github_tree >}}/samples/httpbin) サービスを停止します：

{{< text bash >}}
$ kubectl delete telemetry curl-logging
$ kubectl delete -f @samples/curl/curl.yaml@
$ kubectl delete -f @samples/httpbin/httpbin.yaml@
$ kubectl delete -f @samples/open-telemetry/otel.yaml@ -n istio-system
{{< /text >}}

### Envoy のアクセスログ記録を無効化する {#disable-envoy-access-logging}

Istio インストール設定から `meshConfig.extensionProviders` および
`meshConfig.defaultProviders` の設定を削除するか、`""` に設定してください。

{{< tip >}}
以下の例では、`default` を Istio インストール時に使用したプロファイル名に置き換えてください。
{{< /tip >}}

{{< text bash >}}
$ istioctl install --set profile=default
✔ Istio core installed
✔ Istiod installed
✔ Ingress gateways installed
✔ Installation complete
{{< /text >}}
