---
title: セキュアゲートウェイ
description: TLS または mTLS でサービスをサービスメッシュ外部に公開する方法。
weight: 20
aliases:
  - /zh/docs/tasks/traffic-management/ingress/secure-ingress-sds/
  - /zh/docs/tasks/traffic-management/ingress/secure-ingress-mount/
keywords: [traffic-management, ingress, sds-credentials]
owner: istio/wg-networking-maintainers
test: yes
---

[Ingress トラフィック制御タスク](/ja/docs/tasks/traffic-management/ingress/ingress-control) では、エントリーゲートウェイを構成して HTTP サービスを外部に公開する方法を説明しました。このタスクでは、TLS または mTLS を使ってセキュアな HTTPS サービスを公開する方法を説明します。

{{< boilerplate gateway-api-support >}}

## 準備 {#before-you-begin}

- [インストールガイド](/ja/docs/setup/) を参照して Istio をデプロイしてください。

- [httpbin]({{< github_tree >}}/samples/httpbin) サンプルをデプロイします：

  {{< text bash >}}
  $ kubectl apply -f @samples/httpbin/httpbin.yaml@
  {{< /text >}}

- macOS ユーザーは、[LibreSSL](http://www.libressl.org) ライブラリでビルドされた `curl` を使用しているか確認してください：

  {{< text bash >}}
  $ curl --version | grep LibreSSL
  curl 7.54.0 (x86_64-apple-darwin17.0) libcurl/7.54.0 LibreSSL/2.0.20 zlib/1.2.11 nghttp2/1.24.0
  {{< /text >}}

  上記のような LibreSSL バージョンが表示されれば、このタスクの手順通りに `curl` コマンドが動作します。
  そうでない場合は、Linux マシンなど他の `curl` 実装をお試しください。

## クライアントおよびサーバー証明書と鍵の生成 {#generate-client-and-server-certificates-and-keys}

このタスクでは、お好みのツールで証明書と鍵を生成できます。以下のコマンドは [openssl](https://man.openbsd.org/openssl.1) を使用しています。

1. サービス署名用のルート証明書と秘密鍵を作成します：

   {{< text bash >}}
   $ mkdir example_certs1
   $ openssl req -x509 -sha256 -nodes -days 365 -newkey rsa:2048 -subj '/O=example Inc./CN=example.com' -keyout example_certs1/example.com.key -out example_certs1/example.com.crt
   {{< /text >}}

1. `httpbin.example.com` 用の証明書と秘密鍵を作成します：

   {{< text bash >}}
   $ openssl req -out example_certs1/httpbin.example.com.csr -newkey rsa:2048 -nodes -keyout example_certs1/httpbin.example.com.key -subj "/CN=httpbin.example.com/O=httpbin organization"
   $ openssl x509 -req -sha256 -days 365 -CA example_certs1/example.com.crt -CAkey example_certs1/example.com.key -set_serial 0 -in example_certs1/httpbin.example.com.csr -out example_certs1/httpbin.example.com.crt
   {{< /text >}}

1. 2 組目の同様の証明書と鍵を作成します：

   {{< text bash >}}
   $ mkdir example_certs2
   $ openssl req -x509 -sha256 -nodes -days 365 -newkey rsa:2048 -subj '/O=example Inc./CN=example.com' -keyout example_certs2/example.com.key -out example_certs2/example.com.crt
   $ openssl req -out example_certs2/httpbin.example.com.csr -newkey rsa:2048 -nodes -keyout example_certs2/httpbin.example.com.key -subj "/CN=httpbin.example.com/O=httpbin organization"
   $ openssl x509 -req -sha256 -days 365 -CA example_certs2/example.com.crt -CAkey example_certs2/example.com.key -set_serial 0 -in example_certs2/httpbin.example.com.csr -out example_certs2/httpbin.example.com.crt
   {{< /text >}}

1. `helloworld.example.com` 用の証明書と秘密鍵を作成します：

   {{< text bash >}}
   $ openssl req -out example_certs1/helloworld.example.com.csr -newkey rsa:2048 -nodes -keyout example_certs1/helloworld.example.com.key -subj "/CN=helloworld.example.com/O=helloworld organization"
   $ openssl x509 -req -sha256 -days 365 -CA example_certs1/example.com.crt -CAkey example_certs1/example.com.key -set_serial 1 -in example_certs1/helloworld.example.com.csr -out example_certs1/helloworld.example.com.crt
   {{< /text >}}

1. クライアント証明書と秘密鍵を作成します：

   {{< text bash >}}
   $ openssl req -out example_certs1/client.example.com.csr -newkey rsa:2048 -nodes -keyout example_certs1/client.example.com.key -subj "/CN=client.example.com/O=client organization"
   $ openssl x509 -req -sha256 -days 365 -CA example_certs1/example.com.crt -CAkey example_certs1/example.com.key -set_serial 1 -in example_certs1/client.example.com.csr -out example_certs1/client.example.com.crt
   {{< /text >}}

{{< tip >}}
以下のコマンドで必要なファイルが揃っているか確認できます：

{{< text bash >}}
$ ls example_cert\*
example_certs1:
client.example.com.crt example.com.key httpbin.example.com.crt
client.example.com.csr helloworld.example.com.crt httpbin.example.com.csr
client.example.com.key helloworld.example.com.csr httpbin.example.com.key
example.com.crt helloworld.example.com.key

example_certs2:
example.com.crt httpbin.example.com.crt httpbin.example.com.key
example.com.key httpbin.example.com.csr
{{< /text >}}

{{< /tip >}}

### 単一ホスト TLS エントリーゲートウェイの構成 {#configure-a-tls-ingress-gateway-for-a-single-host}

1. エントリーゲートウェイの Secret を作成します：

   {{< text bash >}}
   $ kubectl create -n istio-system secret tls httpbin-credential \
    --key=example_certs1/httpbin.example.com.key \
    --cert=example_certs1/httpbin.example.com.crt
   {{< /text >}}

1. エントリーゲートウェイを構成します：

{{< tabset category-name="config-api" >}}

{{< tab name="Istio APIs" category-value="istio-apis" >}}

まず、443 ポートのゲートウェイを `servers:` で定義し、`credentialName` の値を `httpbin-credential` に設定します。この値は Secret の名前と同じです。TLS モードの値は `SIMPLE` である必要があります。

{{< text bash >}}
$ cat <<EOF | kubectl apply -f -
apiVersion: networking.istio.io/v1
kind: Gateway
metadata:
name: mygateway
spec:
selector:
istio: ingressgateway # 使用 istio 默认入口网关
servers:

- port:
  number: 443
  name: https
  protocol: HTTPS
  tls:
  mode: SIMPLE
  credentialName: httpbin-credential # 必须与 Secret 相同
  hosts: - httpbin.example.com
  EOF
  {{< /text >}}

次に、ゲートウェイのエントリートラフィックルーティングを構成するために、対応する仮想サービスを定義します：

{{< text bash >}}
$ cat <<EOF | kubectl apply -f -
apiVersion: networking.istio.io/v1
kind: VirtualService
metadata:
name: httpbin
spec:
hosts:

- "httpbin.example.com"
  gateways:
- mygateway
  http:
- match: - uri:
  prefix: /status - uri:
  prefix: /delay
  route: - destination:
  port:
  number: 8000
  host: httpbin
  EOF
  {{< /text >}}

最後に、[これらの説明](/ja/docs/tasks/traffic-management/ingress/ingress-control/#determining-the-ingress-ip-and-ports)
に従って、ゲートウェイの `INGRESS_HOST` と `SECURE_INGRESS_PORT` 変数を設定します。

{{< /tab >}}

{{< tab name="Gateway API" category-value="gateway-api" >}}

まず、
[Kubernetes Gateway](https://gateway-api.sigs.k8s.io/references/spec/#gateway.networking.k8s.io/v1.Gateway)
を作成します：

{{< text bash >}}
$ cat <<EOF | kubectl apply -f -
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
name: mygateway
namespace: istio-system
spec:
gatewayClassName: istio
listeners:

- name: https
  hostname: "httpbin.example.com"
  port: 443
  protocol: HTTPS
  tls:
  mode: Terminate
  certificateRefs: - name: httpbin-credential
  allowedRoutes:
  namespaces:
  from: Selector
  selector:
  matchLabels:
  kubernetes.io/metadata.name: default
  EOF
  {{< /text >}}

次に、ゲートウェイのエントリートラフィックルーティングを構成するために、対応する `HTTPRoute` を定義します：

{{< text bash >}}
$ cat <<EOF | kubectl apply -f -
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
name: httpbin
spec:
parentRefs:

- name: mygateway
  namespace: istio-system
  hostnames: ["httpbin.example.com"]
  rules:
- matches: - path:
  type: PathPrefix
  value: /status - path:
  type: PathPrefix
  value: /delay
  backendRefs: - name: httpbin
  port: 8000
  EOF
  {{< /text >}}

最後に、ゲートウェイのアドレスとポートを `Gateway` リソースから取得します：

{{< text bash >}}
$ kubectl wait --for=condition=programmed gtw mygateway -n istio-system
$ export INGRESS_HOST=$(kubectl get gtw mygateway -n istio-system -o jsonpath='{.status.addresses[0].value}')
$ export SECURE_INGRESS_PORT=$(kubectl get gtw mygateway -n istio-system -o jsonpath='{.spec.listeners[?(@.name=="https")].port}')
{{< /text >}}

{{< /tab >}}

{{< /tabset >}}

3.  HTTPS リクエストを `httpbin` サービスに送信します：

    {{< text bash >}}
    $ curl -v -HHost:httpbin.example.com --resolve "httpbin.example.com:$SECURE_INGRESS_PORT:$INGRESS_HOST" \
     --cacert example_certs1/example.com.crt "https://httpbin.example.com:$SECURE_INGRESS_PORT/status/418"
    ...
    HTTP/2 418
    ...
    I'm a teapot!
    ...
    {{< /text >}}

    `httpbin` サービスは [418 I'm a Teapot](https://tools.ietf.org/html/rfc7168#section-2.3.3) コードを返します。

1.  ゲートウェイの Secret を削除して、異なる証明書と鍵を使用して再作成することでゲートウェイの資格情報を変更します：

    {{< text bash >}}
    $ kubectl -n istio-system delete secret httpbin-credential
    $ kubectl create -n istio-system secret tls httpbin-credential \
     --key=example_certs2/httpbin.example.com.key \
     --cert=example_certs2/httpbin.example.com.crt
    {{< /text >}}

1.  新しい証明書チェーンと `curl` を使用して `httpbin` サービスにアクセスします：

    {{< text bash >}}
    $ curl -v -HHost:httpbin.example.com --resolve "httpbin.example.com:$SECURE_INGRESS_PORT:$INGRESS_HOST" \
     --cacert example_certs2/example.com.crt "https://httpbin.example.com:$SECURE_INGRESS_PORT/status/418"
    ...
    HTTP/2 418
    ...
    I'm a teapot!
    ...
    {{< /text >}}

1.  以前の証明書チェーンを使用して `httpbin` にアクセスした場合は失敗します：

    {{< text bash >}}
    $ curl -v -HHost:httpbin.example.com --resolve "httpbin.example.com:$SECURE_INGRESS_PORT:$INGRESS_HOST" \
     --cacert example_certs1/example.com.crt "https://httpbin.example.com:$SECURE_INGRESS_PORT/status/418"
    ...

    - TLSv1.2 (OUT), TLS handshake, Client hello (1):
    - TLSv1.2 (IN), TLS handshake, Server hello (2):
    - TLSv1.2 (IN), TLS handshake, Certificate (11):
    - TLSv1.2 (OUT), TLS alert, Server hello (2):
    - curl: (35) error:04FFF06A:rsa routines:CRYPTO_internal:block type is not 01
      {{< /text >}}

### 複数ホスト TLS エントリーゲートウェイの構成 {#configure-a-TLS-ingress-gateway-for-multiple-hosts}

複数のホスト（例：`httpbin.example.com` と `helloworld.example.com`）にエントリーゲートウェイを構成できます。エントリーゲートウェイの構成には、各ホストに対応する一意の資格情報が含まれています。

1. 以前の例で `httpbin` の資格情報を復元するために、Secret を再作成します：

   {{< text bash >}}
   $ kubectl -n istio-system delete secret httpbin-credential
   $ kubectl create -n istio-system secret tls httpbin-credential \
    --key=example_certs1/httpbin.example.com.key \
    --cert=example_certs1/httpbin.example.com.crt
   {{< /text >}}

1. `helloworld-v1` サンプルを起動します：

   {{< text bash >}}
   $ kubectl apply -f @samples/helloworld/helloworld.yaml@ -l service=helloworld
   $ kubectl apply -f @samples/helloworld/helloworld.yaml@ -l version=v1
   {{< /text >}}

1. `helloworld-credential` Secret を作成します：

   {{< text bash >}}
   $ kubectl create -n istio-system secret tls helloworld-credential \
    --key=example_certs1/helloworld.example.com.key \
    --cert=example_certs1/helloworld.example.com.crt
   {{< /text >}}

1. `httpbin.example.com` と `helloworld.example.com` ホストを使用してエントリーゲートウェイを構成します：

{{< tabset category-name="config-api" >}}

{{< tab name="Istio APIs" category-value="istio-apis" >}}

443 ポートのゲートウェイを 2 つのサーバーセクションで定義します。各ポートの `credentialName`
値をそれぞれ `httpbin-credential` と `helloworld-credential` に設定します。TLS モードを `SIMPLE` に設定します。

{{< text bash >}}
$ cat <<EOF | kubectl apply -f -
apiVersion: networking.istio.io/v1
kind: Gateway
metadata:
name: mygateway
spec:
selector:
istio: ingressgateway # 使用 istio 默认入口网关
servers:

- port:
  number: 443
  name: https-httpbin
  protocol: HTTPS
  tls:
  mode: SIMPLE
  credentialName: httpbin-credential
  hosts:
  - httpbin.example.com
- port:
  number: 443
  name: https-helloworld
  protocol: HTTPS
  tls:
  mode: SIMPLE
  credentialName: helloworld-credential
  hosts: - helloworld.example.com
  EOF
  {{< /text >}}

ゲートウェイのトラフィックルーティングを構成するために、対応する仮想サービスを定義します。

{{< text bash >}}
$ cat <<EOF | kubectl apply -f -
apiVersion: networking.istio.io/v1
kind: VirtualService
metadata:
name: helloworld
spec:
hosts:

- helloworld.example.com
  gateways:
- mygateway
  http:
- match: - uri:
  exact: /hello
  route: - destination:
  host: helloworld
  port:
  number: 5000
  EOF
  {{< /text >}}

{{< /tab >}}

{{< tab name="Gateway API" category-value="gateway-api" >}}

443 ポートの `Gateway` を 2 つのリスナーで構成します。各リスナーの `certificateRefs`
の名前をそれぞれ `httpbin-credential` と `helloworld-credential` に設定します。

{{< text bash >}}
$ cat <<EOF | kubectl apply -f -
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
name: mygateway
namespace: istio-system
spec:
gatewayClassName: istio
listeners:

- name: https-httpbin
  hostname: "httpbin.example.com"
  port: 443
  protocol: HTTPS
  tls:
  mode: Terminate
  certificateRefs:
  - name: httpbin-credential
    allowedRoutes:
    namespaces:
    from: Selector
    selector:
    matchLabels:
    kubernetes.io/metadata.name: default
- name: https-helloworld
  hostname: "helloworld.example.com"
  port: 443
  protocol: HTTPS
  tls:
  mode: Terminate
  certificateRefs: - name: helloworld-credential
  allowedRoutes:
  namespaces:
  from: Selector
  selector:
  matchLabels:
  kubernetes.io/metadata.name: default
  EOF
  {{< /text >}}

`helloworld` サービスのゲートウェイのトラフィックルーティングを構成します：

{{< text bash >}}
$ cat <<EOF | kubectl apply -f -
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
name: helloworld
spec:
parentRefs:

- name: mygateway
  namespace: istio-system
  hostnames: ["helloworld.example.com"]
  rules:
- matches: - path:
  type: Exact
  value: /hello
  backendRefs: - name: helloworld
  port: 5000
  EOF
  {{< /text >}}

{{< /tab >}}

{{< /tabset >}}

5. HTTPS リクエストを `helloworld.example.com` に送信します：

   {{< text bash >}}
   $ curl -v -HHost:helloworld.example.com --resolve "helloworld.example.com:$SECURE_INGRESS_PORT:$INGRESS_HOST" \
    --cacert example_certs1/example.com.crt "https://helloworld.example.com:$SECURE_INGRESS_PORT/hello"
   ...
   HTTP/2 200
   ...
   {{< /text >}}

1. HTTPS リクエストを `httpbin.example.com` に送信し、まだ [HTTP 418](https://datatracker.ietf.org/doc/html/rfc2324) を返します：

   {{< text bash >}}
   $ curl -v -HHost:httpbin.example.com --resolve "httpbin.example.com:$SECURE_INGRESS_PORT:$INGRESS_HOST" \
    --cacert example_certs1/example.com.crt "https://httpbin.example.com:$SECURE_INGRESS_PORT/status/418"
   ...
   HTTP/2 418
   ...
   server: istio-envoy
   ...
   {{< /text >}}

### 双方向 TLS エントリーゲートウェイの構成 {#configure-a-mutual-tls-ingress-gateway}

ゲートウェイの定義を拡張して [双方向 TLS](https://en.wikipedia.org/wiki/Mutual_authentication) をサポートできます。

1. その Secret を削除して新しいものを作成することでエントリーゲートウェイの資格情報を変更します。サーバーは CA 証明書を使用してクライアントを検証し、CA 証明書を保持するために名前 `ca.crt` を使用する必要があります。

   {{< text bash >}}
   $ kubectl -n istio-system delete secret httpbin-credential
   $ kubectl create -n istio-system secret generic httpbin-credential \
    --from-file=tls.key=example_certs1/httpbin.example.com.key \
    --from-file=tls.crt=example_certs1/httpbin.example.com.crt \
    --from-file=ca.crt=example_certs1/example.com.crt
   {{< /text >}}

   {{< tip >}}

   {{< boilerplate crl-tip >}}

   資格情報には [OCSP Staple](https://datatracker.ietf.org/doc/html/rfc6961) も含めることができ、パラメータ `--from-file=tls.ocsp-staple=/some/path/to/your-ocsp-staple.pem` で指定された `tls.ocsp-staple` をキーとして使用できます。

   {{< /tip >}}

1. エントリーゲートウェイを構成します：

{{< tabset category-name="config-api" >}}

{{< tab name="Istio APIs" category-value="istio-apis" >}}

ゲートウェイの定義を変更して TLS モードを `MUTUAL` に設定します。

{{< text bash >}}
$ cat <<EOF | kubectl apply -f -
apiVersion: networking.istio.io/v1
kind: Gateway
metadata:
name: mygateway
spec:
selector:
istio: ingressgateway # 使用 istio 默认入口网关
servers:

- port:
  number: 443
  name: https
  protocol: HTTPS
  tls:
  mode: MUTUAL
  credentialName: httpbin-credential # 必须与 Secret 相同
  hosts: - httpbin.example.com
  EOF
  {{< /text >}}

{{< /tab >}}

{{< tab name="Gateway API" category-value="gateway-api" >}}

Kubernetes Gateway API は現在、[Gateway](https://gateway-api.sigs.k8s.io/references/spec/#gateway.networking.k8s.io/v1.Gateway)
の双方向 TLS 終端をサポートしていないため、Istio 固有のオプション `gateway.istio.io/tls-terminate-mode: MUTUAL` を使用して構成します：

{{< text bash >}}
$ cat <<EOF | kubectl apply -f -
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
name: mygateway
namespace: istio-system
spec:
gatewayClassName: istio
listeners:

- name: https
  hostname: "httpbin.example.com"
  port: 443
  protocol: HTTPS
  tls:
  mode: Terminate
  certificateRefs: - name: httpbin-credential
  options:
  gateway.istio.io/tls-terminate-mode: MUTUAL
  allowedRoutes:
  namespaces:
  from: Selector
  selector:
  matchLabels:
  kubernetes.io/metadata.name: default
  EOF
  {{< /text >}}

{{< /tab >}}

{{< /tabset >}}

3. 以前の方法で HTTPS リクエストを送信してみて、どのように失敗するかを確認します：

   {{< text bash >}}
   $ curl -v -HHost:httpbin.example.com --resolve "httpbin.example.com:$SECURE_INGRESS_PORT:$INGRESS_HOST" \
    --cacert example_certs1/example.com.crt "https://httpbin.example.com:$SECURE_INGRESS_PORT/status/418"

   - TLSv1.3 (OUT), TLS handshake, Client hello (1):
   - TLSv1.3 (IN), TLS handshake, Server hello (2):
   - TLSv1.3 (IN), TLS handshake, Encrypted Extensions (8):
   - TLSv1.3 (IN), TLS handshake, Request CERT (13):
   - TLSv1.3 (IN), TLS handshake, Certificate (11):
   - TLSv1.3 (IN), TLS handshake, CERT verify (15):
   - TLSv1.3 (IN), TLS handshake, Finished (20):
   - TLSv1.3 (OUT), TLS change cipher, Change cipher spec (1):
   - TLSv1.3 (OUT), TLS handshake, Certificate (11):
   - TLSv1.3 (OUT), TLS handshake, Finished (20):
   - TLSv1.3 (IN), TLS alert, unknown (628):
   - OpenSSL SSL_read: error:1409445C:SSL routines:ssl3_read_bytes:tlsv13 alert certificate required, errno 0
     {{< /text >}}

1. クライアント証明書と秘密鍵を `curl` に渡して再送信します。クライアント証明書を `--cert` フラグで、秘密鍵を `--key` フラグで `curl` に渡します：

   {{< text bash >}}
   $ curl -v -HHost:httpbin.example.com --resolve "httpbin.example.com:$SECURE_INGRESS_PORT:$INGRESS_HOST" \
    --cacert example_certs1/example.com.crt --cert example_certs1/client.example.com.crt --key example_certs1/client.example.com.key \
    "https://httpbin.example.com:$SECURE_INGRESS_PORT/status/418"
   ...
   HTTP/2 418
   ...
   server: istio-envoy
   ...
   I'm a teapot!
   ...
   {{< /text >}}

## より詳しい情報 {#more-info}

### 鍵の形式 {#key-formats}

Istio は、[cert-manager](/ja/docs/ops/integrations/certmanager/) などのさまざまなツールとの統合をサポートするために、いくつかの異なる Secret 形式を読み取ることができます：

- TLS Secret は `tls.key` と `tls.crt` を含み、上記のように双方向 TLS の場合は `ca.crt` をキーとして使用できます。
- TLS Secret は `tls.key` と `tls.crt` を含みます。
  双方向 TLS の場合、単一の一般的な Secret は `<secret>-cacert` という名前で、`cacert` キーを持ちます。
  例：`httpbin-credential` は `tls.key` と `tls.crt` を持ち、`httpbin-credential-cacert` は `cacert` を持ちます。
- 一般的な Secret は `key` と `cert` キーを持ちます。双方向 TLS の場合、`cacert` をキーとして使用できます。
- 一般的な Secret は `key` と `cert` キーを持ちます。双方向 TLS の場合、名前 `<secret>-cacert` を持つ一般的な Secret は `cacert` キーを持ちます。
  例：`httpbin-credential` は `key` と `cert` を持ち、`httpbin-credential-cacert` は `cacert` を持ちます。
- `cacert` キーの値は、連結された各 CA 証明書からなる CA バンドルである可能性があります。

### SNI ルーティング {#sni-routing}

HTTPS `Gateway` は、リクエストを転送する前に構成されたホストに対して [SNI](https://en.wikipedia.org/wiki/Server_Name_Indication)
マッチングを実行するため、一部のリクエストが失敗する可能性があります。詳細については、
[SNI ルーティングの構成](/ja/docs/ops/common-problems/network-issues/#configuring-sni-routing-when-not-sending-sni) を参照してください。

## トラブルシューティング {#troubleshooting}

- `INGRESS_HOST` と `SECURE_INGRESS_PORT` 環境変数の値を確認してください。以下のコマンドの出力により、有効な値であることを確認してください：

  {{< text bash >}}
  $ kubectl get svc -n istio-system
  $ echo "INGRESS_HOST=$INGRESS_HOST, SECURE_INGRESS_PORT=$SECURE_INGRESS_PORT"
  {{< /text >}}

- `INGRESS_HOST` の値が IP アドレスであることを確認してください。一部のクラウドプラットフォーム（例：AWS）では、ドメイン名ではなく IP アドレスが得られる可能性があります。
  このタスクでは IP アドレスが必要なため、以下のようなコマンドを使用して変換する必要があります：

  {{< text bash >}}
  $ nslookup ab52747ba608744d8afd530ffd975cbf-330887905.us-east-1.elb.amazonaws.com
  $ export INGRESS_HOST=3.225.207.109
  {{< /text >}}

- ゲートウェイコントローラーのログを確認してエラーメッセージを取得してください：

  {{< text syntax=bash snip_id=none >}}
  $ kubectl logs -n istio-system <gateway-service-pod>
  {{< /text >}}

- macOS を使用している場合は、[準備](#before-you-begin) セクションで説明したように、[LibreSSL](http://www.libressl.org/) `curl`
  ライブラリでビルドされた `curl` を使用しているか確認してください。

- `istio-system` 名前空間に成功して Secret が作成されたことを確認してください：

  {{< text bash >}}
  $ kubectl -n istio-system get secrets
  {{< /text >}}

  `httpbin-credential` と `helloworld-credential` が Secret リストに表示されるはずです。

- エントリーゲートウェイプロキシーがゲートウェイに鍵/証明書ペアをプッシュしたことを確認するために、ログを確認してください：

  {{< text syntax=bash snip_id=none >}}
  $ kubectl logs -n istio-system <gateway-service-pod>
  {{< /text >}}

  ログには `httpbin-credential` Secret が追加されているはずです。双方向 TLS を使用している場合は、`httpbin-credential-cacert` Secret も表示されるはずです。
  ログにはゲートウェイプロキシーがエントリーゲートウェイから SDS リクエストを受信し、リソース名が `httpbin-credential` であり、エントリーゲートウェイが鍵/証明書ペアを取得したことが表示されます。双方向 TLS を使用している場合、ログには鍵/証明書がエントリーゲートウェイに送信され、エントリーゲートウェイが `httpbin-credential-cacert` リソース名の SDS リクエストを受信し、ルート証明書を取得したことが表示されます。

## クリーンアップ {#cleanup}

1. ゲートウェイ構成とルーティングを削除します：

{{< tabset category-name="config-api" >}}

{{< tab name="Istio APIs" category-value="istio-apis" >}}

{{< text bash >}}
$ kubectl delete gateway mygateway
$ kubectl delete virtualservice httpbin helloworld
{{< /text >}}

{{< /tab >}}

{{< tab name="Gateway API" category-value="gateway-api" >}}

{{< text bash >}}
$ kubectl delete -n istio-system gtw mygateway
$ kubectl delete httproute httpbin helloworld
{{< /text >}}

{{< /tab >}}

{{< /tabset >}}

2. Secret、証明書、および鍵を削除します：

   {{< text bash >}}
   $ kubectl delete -n istio-system secret httpbin-credential helloworld-credential
   $ rm -rf ./example_certs1 ./example_certs2
   {{< /text >}}

1. `httpbin` と `helloworld` サービスを停止します：

   {{< text bash >}}
   $ kubectl delete -f samples/httpbin/httpbin.yaml
   $ kubectl delete deployment helloworld-v1
   $ kubectl delete service helloworld
   {{< /text >}}
