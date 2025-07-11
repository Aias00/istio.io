---
title: Gateway ネットワークトポロジの設定
description: Gateway ネットワークトポロジの設定方法。
weight: 60
keywords: [traffic-management, ingress, gateway]
owner: istio/wg-networking-maintainers
test: yes
status: Alpha
---

{{< boilerplate alpha >}}

{{< boilerplate gateway-api-support >}}

## 外部クライアント属性（IP アドレス、証明書情報）を宛先ワークロードに転送する {#forwarding-external-client-attributes-to-destination-workloads}

多くのアプリケーションは、リクエスト元クライアントの IP アドレスや証明書情報を知る必要があります。
これには、クライアント IP を記録するログツールや監査ツール、
この情報を使ってルールセットを正しく適用する必要があるセキュリティツール（Web アプリケーションファイアウォールなど）が含まれます。
サービスにクライアント属性を提供する機能は、リバースプロキシの基本的な役割の 1 つです。
これらのクライアント属性を宛先ワークロードに転送するために、
プロキシは `X-Forwarded-For`（XFF）や `X-Forwarded-Client-Cert`（XFCC）リクエストヘッダーを利用できます。

現代のネットワークは多様ですが、ネットワークトポロジに関係なく、これらの属性のサポートは必要です。
クラウドベースのロードバランサ、オンプレミスのロードバランサ、インターネットに直接公開された Gateway、
多くの中間プロキシを経由する Gateway、その他の未定義のデプロイメントトポロジでも、
これらの情報は保存・転送される必要があります。

Istio は[イングレスゲートウェイ](/ja/docs/tasks/traffic-management/ingress/ingress-control/)を提供していますが、
上記のような多様なアーキテクチャの複雑さを考慮すると、クライアント属性を宛先ワークロードに正しく転送するための合理的なデフォルト値を提供するのは困難です。
Istio のマルチクラスター展開が一般的になるにつれ、この問題の重要性は増しています。

`X-Forwarded-For` についての詳細は IETF の [RFC](https://tools.ietf.org/html/rfc7239) を参照してください。

## ネットワークトポロジの設定 {#configuring-network-topologies}

XFF および XFCC リクエストヘッダーの設定は、`MeshConfig` で全 Gateway ワークロードに対してグローバルに設定することも、
Pod アノテーションで各 Gateway ごとに設定することもできます。たとえば、インストールやアップグレード時に `IstioOperator` カスタムリソースでグローバル設定を行います：

{{< text syntax=yaml snip_id=none >}}
spec:
meshConfig:
defaultConfig:
gatewayTopology:
numTrustedProxies: <VALUE>
forwardClientCertDetails: <ENUM_VALUE>
{{< /text >}}

Istio イングレスゲートウェイ Pod の spec で `proxy.istio.io/config` アノテーションを追加しても、これらの設定が可能です。

{{< text syntax=yaml snip_id=none >}}
...
metadata:
annotations:
"proxy.istio.io/config": '{"gatewayTopology" : { "numTrustedProxies": <VALUE>, "forwardClientCertDetails": <ENUM_VALUE> } }'
{{< /text >}}

### X-Forwarded-For ヘッダーの設定 {#configuring-X-Forwarded-For-headers}

アプリケーションはリバースプロキシを通じて、`X-Forwarded-For` ヘッダーなどのクライアント属性を受け取ります。
しかし、Istio は多様なネットワークトポロジでデプロイできるため、
Istio ゲートウェイプロキシの上流にある信頼できるプロキシの数 `numTrustedProxies` を設定する必要があります。
これにより、クライアントアドレスが正しく抽出されます。
この設定は、イングレスゲートウェイが `X-Envoy-External-Address` ヘッダーに格納する値を制御し、
上流サービスがクライアントの元 IP アドレスを信頼して利用できるようにします。

たとえば、Istio Gateway の前にクラウドロードバランサとリバースプロキシがある場合、`numTrustedProxies` を `2` に設定します。

{{< idea >}}
Istio Gateway プロキシの前にあるすべてのプロキシは、HTTP トラフィックを解釈し、
各転送時に `X-Forwarded-For` ヘッダーに情報を追加する必要があります。
`X-Forwarded-For` ヘッダーのエントリ数が設定した信頼できるホップ数より少ない場合、
Envoy は下流アドレスを信頼できるクライアントアドレスとして使用します。
`X-Forwarded-For` ヘッダーと信頼できるクライアントアドレスの決定方法については [Envoy ドキュメント](https://www.envoyproxy.io/docs/envoy/latest/configuration/http/http_conn_man/headers#x-forwarded-for) を参照してください。
{{< /idea >}}

#### httpbin X-Forwarded-For の例 {#example-using-X-Forwarded-For-capability-with-httpbin}

1.  `topology.yaml` ファイルを作成し、`numTrustedProxies` を `2` に設定して Istio をインストールします：

    {{< text syntax=bash snip_id=install_num_trusted_proxies_two >}}
    $ cat <<EOF > topology.yaml
    apiVersion: install.istio.io/v1alpha1
    kind: IstioOperator
    spec:
    meshConfig:
    defaultConfig:
    gatewayTopology:
    numTrustedProxies: 2
    EOF
    $ istioctl install -f topology.yaml
    {{< /text >}}

    {{< idea >}}
    すでに Istio イングレスゲートウェイをインストールしている場合は、1 の後にすべてのイングレスゲートウェイ Pod を再起動してください。
    {{</ idea >}}

1.  `httpbin` ネームスペースを作成します：

    {{< text syntax=bash snip_id=create_httpbin_namespace >}}
    $ kubectl create namespace httpbin
    namespace/httpbin created
    {{< /text >}}

1.  Sidecar インジェクションを有効化し、`istio-injection` ラベルを `enabled` に設定します：

    {{< text syntax=bash snip_id=label_httpbin_namespace >}}
    $ kubectl label --overwrite namespace httpbin istio-injection=enabled
    namespace/httpbin labeled
    {{< /text >}}

1.  `httpbin` を `httpbin` ネームスペースにデプロイします：

    {{< text syntax=bash snip_id=apply_httpbin >}}
    $ kubectl apply -n httpbin -f @samples/httpbin/httpbin.yaml@
    {{< /text >}}

1.  `httpbin` 用の Gateway をデプロイします：

{{< tabset category-name="config-api" >}}

{{< tab name="Istio APIs" category-value="istio-apis" >}}

{{< text syntax=bash snip_id=deploy_httpbin_gateway >}}
$ kubectl apply -n httpbin -f @samples/httpbin/httpbin-gateway.yaml@
{{< /text >}}

{{< /tab >}}

{{< tab name="Gateway API" category-value="gateway-api" >}}

{{< text syntax=bash snip_id=deploy_httpbin_k8s_gateway >}}
$ kubectl apply -n httpbin -f @samples/httpbin/gateway-api/httpbin-gateway.yaml@
$ kubectl wait --for=condition=programmed gtw -n httpbin httpbin-gateway
{{< /text >}}

{{< /tab >}}

{{< /tabset >}}

6. Istio イングレスゲートウェイに基づいてローカル環境変数 `GATEWAY_URL` を設定します：

{{< tabset category-name="config-api" >}}

{{< tab name="Istio APIs" category-value="istio-apis" >}}

{{< text syntax=bash snip_id=export_gateway_url >}}
$ export GATEWAY_URL=$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
{{< /text >}}

{{< /tab >}}

{{< tab name="Gateway API" category-value="gateway-api" >}}

{{< text syntax=bash snip_id=export_k8s_gateway_url >}}
$ export GATEWAY_URL=$(kubectl get gateways.gateway.networking.k8s.io httpbin-gateway -n httpbin -ojsonpath='{.status.addresses[0].value}')
{{< /text >}}

{{< /tab >}}

{{< /tabset >}}

7. 次の `curl` コマンドを実行し、`X-Forwarded-For` ヘッダーにプロキシアドレスを含めてリクエストを送信します：

   {{< text syntax=bash snip_id=curl_xff_headers >}}
   $ curl -s -H 'X-Forwarded-For: 56.5.6.7, 72.9.5.6, 98.1.2.3' "$GATEWAY_URL/get?show_env=true" | jq '.headers["X-Forwarded-For"][0]'
   "56.5.6.7, 72.9.5.6, 98.1.2.3,10.244.0.1"
   {{< /text >}}

{{< tip >}}
上記の例では、`$GATEWAY_URL` は 10.244.0.1 に解決されています。これはご利用の環境によって異なる場合があります。
{{< /tip >}}

上記の出力は、`httpbin` ワークロードが受け取ったリクエストヘッダーを示しています。Istio Gateway がこのリクエストを受信すると、
`X-Envoy-External-Address` ヘッダーを curl コマンドの `X-Forwarded-For` ヘッダーの後ろから 2 番目のアドレス（`numTrustedProxies: 2`）に設定します。
さらに、Gateway は自身の IP を `X-Forwarded-For` ヘッダーに追加してから `httpbin` ワークロードに転送します。

### X-Forwarded-Client-Cert ヘッダーの設定 {#configuring-X-Forwarded-Client-Cert-headers}

[Envoy ドキュメント](https://www.envoyproxy.io/docs/envoy/latest/configuration/http/http_conn_man/headers#x-forwarded-client-cert) から XFCC の説明：

{{< quote >}}
x-forwarded-client-cert（XFCC）はプロキシリクエストヘッダーであり、
リクエストがクライアントからサーバーに流れる途中で経由した一部またはすべてのクライアントおよびプロキシの証明書情報を示します。
プロキシは、リクエストをプロキシする前に XFCC ヘッダーをクリア・追加・転送するかを選択できます。
{{< /quote >}}

XFCC ヘッダーの処理方法を設定するには、`IstioOperator` で `forwardClientCertDetails` を設定します：

{{< text syntax=yaml snip_id=none >}}
apiVersion: install.istio.io/v1alpha1
kind: IstioOperator
spec:
meshConfig:
defaultConfig:
gatewayTopology:
forwardClientCertDetails: <ENUM_VALUE>
{{< /text >}}

`ENUM_VALUE` には以下の値を指定できます：

| `ENUM_VALUE`          |                                                                                                                                       |
| --------------------- | ------------------------------------------------------------------------------------------------------------------------------------- |
| `UNDEFINED`           | フィールドが設定されていません。                                                                                                      |
| `SANITIZE`            | XFCC ヘッダーを次のホップに送信しません。                                                                                             |
| `FORWARD_ONLY`        | クライアント接続が mTLS（Mutual TLS）の場合、リクエストで XFCC ヘッダーを転送します。                                                 |
| `APPEND_FORWARD`      | クライアント接続が mTLS の場合、クライアント証明書情報を XFCC ヘッダーに追加して転送します。                                          |
| `SANITIZE_SET`        | クライアント接続が mTLS の場合、XFCC ヘッダーをクライアント証明書情報でリセットし、次のホップに送信します（Gateway のデフォルト値）。 |
| `ALWAYS_FORWARD_ONLY` | クライアント接続が mTLS かどうかに関係なく、常にリクエストで XFCC ヘッダーを転送します。                                              |

詳細や利用例は [Envoy ドキュメント](https://www.envoyproxy.io/docs/envoy/latest/configuration/http/http_conn_man/headers#x-forwarded-client-cert) を参照してください。

## PROXY プロトコル {#PROXY-protocol}

[PROXY プロトコル](https://www.haproxy.org/download/1.8/doc/proxy-protocol.txt) は、HTTP や
`X-Forwarded-For`、`X-Envoy-External-Address` などの L7 プロトコルに依存せず、複数の TCP プロキシ間でクライアント属性を交換・保存できます。
このプロトコルは、外部 TCP ロードバランサが Istio Gateway を経由して TCP トラフィックをバックエンド TCP サービスにプロキシし、
クライアント属性（例：送信元 IP）を上流の TCP サービスエンドポイントに公開したい場合に適しています。PROXY プロトコルは `EnvoyFilter` で有効化できます。

{{< warning >}}
Envoy が TCP トラフィックを転送する場合のみ PROXY プロトコルがサポートされます。
詳細や重要なパフォーマンス上の注意点については
[Envoy ドキュメント](https://www.envoyproxy.io/docs/envoy/latest/intro/arch_overview/other_features/ip_transparency#proxy-protocol) を参照してください。

PROXY プロトコルは L7 トラフィックには使用しないでください。また、L7 ロードバランサの後ろで Istio Gateway を使う場合にも使用しないでください。
{{< /warning >}}

外部ロードバランサが TCP トラフィックを転送し、PROXY プロトコルを使用する場合、Istio Gateway の TCP リスナーも PROXY プロトコルを受け入れるように設定する必要があります。
Gateway のすべての TCP リスナーで PROXY プロトコルを有効にするには、`IstioOperator` で `proxyProtocol` を設定します。
例：

{{< text syntax=yaml snip_id=none >}}
apiVersion: install.istio.io/v1alpha1
kind: IstioOperator
spec:
meshConfig:
defaultConfig:
gatewayTopology:
proxyProtocol: {}
{{< /text >}}

また、次の Pod アノテーションを持つ Gateway をデプロイします：

{{< text yaml >}}
metadata:
annotations:
"proxy.istio.io/config": '{"gatewayTopology" : { "proxyProtocol": {} }}'
{{< /text >}}

クライアント IP は Gateway が PROXY プロトコルから取得し、`X-Forwarded-For` および `X-Envoy-External-Address` ヘッダーに設定（または追加）されます。
PROXY プロトコルは `X-Forwarded-For` や `X-Envoy-External-Address` などの L7 リクエストヘッダーとは排他的です。
`gatewayTopology` 設定とともに PROXY プロトコルを使用する場合、信頼できるクライアントアドレスの決定には `numTrustedProxies` と受信した `X-Forwarded-For` ヘッダーが優先され、PROXY プロトコルのクライアント情報は無視されます。

上記の例は Gateway を PROXY プロトコル TCP トラフィックの受け入れにのみ設定しています。
Envoy 自体を上流サービスとの通信で PROXY プロトコルを使うように設定する例は、
[Envoy ドキュメント](https://www.envoyproxy.io/docs/envoy/latest/intro/arch_overview/other_features/ip_transparency#proxy-protocol) を参照してください。
