---
title: Envoy アクセスログの取得
description: このタスクでは、Envoy プロキシがアクセスログを標準出力に出力するように設定する方法を紹介します。
weight: 10
keywords: [telemetry, logs]
aliases:
  - /zh/docs/tasks/telemetry/access-log
  - /zh/docs/tasks/telemetry/logs/access-log/
owner: istio/wg-policies-and-telemetry-maintainers
test: yes
---

Istio で最も基本的なログの種類は
[Envoy のアクセスログ](https://www.envoyproxy.io/docs/envoy/latest/configuration/observability/access_log/usage) です。
Envoy プロキシはアクセス情報を標準出力に出力します。Envoy コンテナの標準出力は `kubectl logs` コマンドで確認できます。

{{< boilerplate before-you-begin-egress >}}

{{< boilerplate start-httpbin-service >}}

## Envoy アクセスログの有効化 {#enable-envoy-s-access-logging}

Istio ではアクセスログを有効化する方法がいくつか用意されており、Telemetry API の利用が推奨されています。

### Telemetry API を使う {#using-telemetry-API}

Telemetry API でアクセスログを有効化または無効化できます：

{{< text yaml >}}
apiVersion: telemetry.istio.io/v1
kind: Telemetry
metadata:
name: mesh-default
namespace: istio-system
spec:
accessLogging: - providers: - name: envoy
{{< /text >}}

上記の例ではデフォルトの `envoy` アクセスログプロバイダを使用しており、デフォルト設定以外は特に指定していません。

同様の設定は個別の名前空間やワークロード単位にも適用でき、きめ細かいログ制御が可能です。

Telemetry API の詳細は [Telemetry API 概要](/zh/docs/tasks/observability/telemetry/) をご覧ください。

### メッシュ設定を使う {#using-mesh-config}

`IstioOperator` 設定で Istio をインストールする場合、次のフィールドを追加してください：

{{< text yaml >}}
spec:
meshConfig:
accessLogFile: /dev/stdout
{{< /text >}}

または、元の `istioctl install` コマンドに同じ設定を追加します。例：

{{< text syntax=bash snip_id=none >}}
$ istioctl install <flags-you-used-to-install-Istio> --set meshConfig.accessLogFile=/dev/stdout
{{< /text >}}

`accessLogEncoding` を `JSON` または `TEXT` に設定することで、2 つのフォーマットを切り替えることもできます。

また、`accessLogFormat` を設定することで、アクセスログの[フォーマット](https://www.envoyproxy.io/docs/envoy/latest/configuration/observability/access_log/usage#format-rules)をカスタマイズできます。

これら 3 つの設定の詳細は [global mesh options](/zh/docs/reference/config/istio.mesh.v1alpha1/#MeshConfig) を参照してください：

- `meshConfig.accessLogFile`
- `meshConfig.accessLogEncoding`
- `meshConfig.accessLogFormat`

## デフォルトのアクセスログフォーマット {#default-access-log-format}

`accessLogFormat` を指定しない場合、Istio は次のデフォルトのアクセスログフォーマットを使用します：

{{< text plain >}}
[%START_TIME%] "%REQ(:METHOD)% %REQ(X-ENVOY-ORIGINAL-PATH?:PATH)% %PROTOCOL%" %RESPONSE_CODE% %RESPONSE_FLAGS% %RESPONSE_CODE_DETAILS% %CONNECTION_TERMINATION_DETAILS%
"%UPSTREAM_TRANSPORT_FAILURE_REASON%" %BYTES_RECEIVED% %BYTES_SENT% %DURATION% %RESP(X-ENVOY-UPSTREAM-SERVICE-TIME)% "%REQ(X-FORWARDED-FOR)%" "%REQ(USER-AGENT)%" "%REQ(X-REQUEST-ID)%"
"%REQ(:AUTHORITY)%" "%UPSTREAM_HOST%" %UPSTREAM_CLUSTER_RAW% %UPSTREAM_LOCAL_ADDRESS% %DOWNSTREAM_LOCAL_ADDRESS% %DOWNSTREAM_REMOTE_ADDRESS% %REQUESTED_SERVER_NAME% %ROUTE_NAME%\n
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
| `%UPSTREAM_CLUSTER_RAW%`                                         | <code>outbound&#124;8000&#124;&#124;httpbin.foo.svc.cluster.local</code> | <code>inbound&#124;8000&#124;&#124;</code>        |
| `%UPSTREAM_LOCAL_ADDRESS%`                                       | `10.44.1.23:37652`                                                       | `127.0.0.1:41854`                                 |
| `%DOWNSTREAM_LOCAL_ADDRESS%`                                     | `10.0.45.184:8000`                                                       | `10.44.1.27:80`                                   |
| `%DOWNSTREAM_REMOTE_ADDRESS%`                                    | `10.44.1.23:46520`                                                       | `10.44.1.23:37652`                                |
| `%REQUESTED_SERVER_NAME%`                                        | `-`                                                                      | `outbound_.8000_._.httpbin.foo.svc.cluster.local` |
| `%ROUTE_NAME%`                                                   | `default`                                                                | `default`                                         |

## アクセスログのテスト {#test-the-access-log}

1. `curl` から `httpbin` へリクエストを送信します：

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

1. `curl` のログを確認します：

   {{< text bash >}}
   $ kubectl logs -l app=curl -c istio-proxy
   [2019-03-06T09:31:27.354Z] "GET /status/418 HTTP/1.1" 418 - "-" 0 135 11 10 "-" "curl/7.60.0" "d209e46f-9ed5-9b61-bbdd-43e22662702a" "httpbin:8000" "172.30.146.73:80" outbound|8000||httpbin.default.svc.cluster.local - 172.21.13.94:8000 172.30.146.82:60290 -
   {{< /text >}}

1. `httpbin` のログを確認します：

   {{< text bash >}}
   $ kubectl logs -l app=httpbin -c istio-proxy
   [2019-03-06T09:31:27.360Z] "GET /status/418 HTTP/1.1" 418 - "-" 0 135 5 2 "-" "curl/7.60.0" "d209e46f-9ed5-9b61-bbdd-43e22662702a" "httpbin:8000" "127.0.0.1:80" inbound|8000|http|httpbin.default.svc.cluster.local - 172.30.146.73:80 172.30.146.82:38618 outbound*.8000*.\_.httpbin.default.svc.cluster.local
   {{< /text >}}

リクエストに対応する情報が、送信元（`curl`）と宛先（`httpbin`）の Istio
プロキシログにそれぞれ出力されていることに注目してください。ログには HTTP メソッド（`GET`）、HTTP パス（`/status/418`）、
レスポンスコード（`418`）やその他の[リクエスト関連情報](https://www.envoyproxy.io/docs/envoy/latest/configuration/observability/access_log/usage#format-rules)が含まれています。

## クリーンアップ {#cleanup}

[curl]({{<github_tree>}}/samples/curl) と
[httpbin]({{<github_tree>}}/samples/httpbin) サービスを停止します：

{{< text bash >}}
$ kubectl delete -f @samples/curl/curl.yaml@
$ kubectl delete -f @samples/httpbin/httpbin.yaml@
{{< /text >}}

### Envoy のアクセスログを無効化する {#disable-envoy-s-access-logging}

`istio` 設定ファイルを編集し、`meshConfig.accessLogFile` を `""` に設定します。

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
