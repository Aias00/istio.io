---
title: Egress ゲートウェイの TLS オリジネーション
description: Egress ゲートウェイを使って外部サービスへの TLS 接続を発行する方法を説明します。
weight: 40
keywords: [traffic-management, egress]
aliases:
  - /zh/docs/examples/advanced-gateways/egress-gateway-tls-origination/
  - /zh/docs/examples/advanced-gateways/egress-gateway-tls-origination-sds/
  - /zh/docs/tasks/traffic-management/egress/egress-gateway-tls-origination-sds/
owner: istio/wg-networking-maintainers
test: yes
---

[出口トラフィックの TLS オリジネーション](/ja/docs/tasks/traffic-management/egress/egress-tls-origination/) の例では、Istio で外部サービスへのトラフィックに {{< gloss >}}TLS オリジネーション{{< /gloss >}} を適用する方法を紹介しました。
[egress ゲートウェイの構成](/ja/docs/tasks/traffic-management/egress/egress-gateway/) の例では、Istio で専用の egress ゲートウェイサービス経由で出口トラフィックを誘導する方法を紹介しました。
本例は両者を組み合わせ、egress ゲートウェイで外部サービスへのトラフィックに TLS オリジネーションを行う方法を説明します。

{{< boilerplate gateway-api-support >}}

## 始める前に{#before-you-begin}

- [インストールガイド](/ja/docs/setup/) の手順に従って Istio をインストールしてください。

- [curl]({{< github_tree >}}/samples/curl) サンプルアプリを起動し、外部リクエストのテストソースとします。

  [Sidecar の自動注入](/ja/docs/setup/additional-setup/sidecar-injection/#automatic-sidecar-injection)が有効な場合は、

  {{< text bash >}}
    $ kubectl apply -f @samples/curl/curl.yaml@
  {{< /text >}}

  そうでない場合は、`curl` アプリをデプロイする前に手動で Sidecar を注入してください：

  {{< text bash >}}
    $ kubectl apply -f <(istioctl kube-inject -f @samples/curl/curl.yaml@)
  {{< /text >}}

  `exec` や `curl` 操作を行う Pod にはすべて Sidecar を注入する必要があります。

- 外部サービスにリクエストを送信するソース Pod の名前を保存する shell 変数を作成します。
  [curl]({{< github_tree >}}/samples/curl) サンプルを使う場合は、次のコマンドを実行します：

  {{< text bash >}}
    $ export SOURCE_POD=$(kubectl get pod -l app=curl -o jsonpath={.items..metadata.name})
  {{< /text >}}

- macOS ユーザーは `openssl` のバージョンが 1.1 以上であることを確認してください：

  {{< text bash >}}
    $ openssl version -a | grep OpenSSL
    OpenSSL 1.1.1g 21 Apr 2020
  {{< /text >}}

  上記コマンドの出力が `1.1` 以上であれば、このタスクの指示通りに `openssl` コマンドが動作します。そうでない場合は、`openssl` をアップグレードするか、Linux マシンなど他の実装を試してください。

- [Envoy のアクセスログを有効化](/ja/docs/tasks/observability/logs/access-log/#enable-envoy-s-access-logging)してください。未有効の場合は、例：

  {{< text bask >}}
    $ istioctl install <flags-you-used-to-install-Istio> --set meshConfig.accessLogFile=/dev/stdout
  {{< /text >}}

- `Gateway API` の手順を使わない場合は、[Istio Egress ゲートウェイのデプロイ](/ja/docs/tasks/traffic-management/egress/egress-gateway/#deploy-istio-egress-gateway)が済んでいることを確認してください。

## Egress ゲートウェイで TLS オリジネーションを行う {#perform-TLS-origination-with-an-egress-gateway}

このセクションでは、[出口トラフィックの TLS オリジネーション](/ja/docs/tasks/traffic-management/egress/egress-tls-origination/) の例と同じ TLS オリジネーションを Egress ゲートウェイで行う方法を説明します。ここでは TLS オリジネーションは Egress ゲートウェイで行われ、前の例のように Sidecar で行うのではありません。

1.  `edition.cnn.com` の `ServiceEntry` を定義します：

    {{< text bash >}}
    $ kubectl apply -f - <<EOF
    apiVersion: networking.istio.io/v1
    kind: ServiceEntry
    metadata:
    name: cnn
    spec:
    hosts:

    - edition.cnn.com
      ports:
    - number: 80
      name: http
      protocol: HTTP
    - number: 443
      name: https
      protocol: HTTPS
      resolution: DNS
      EOF
      {{< /text >}}

1.  [http://edition.cnn.com/politics](https://edition.cnn.com/politics) へのリクエストを送信し、`ServiceEntry` が正しく適用されたことを確認します。

    {{< text bash >}}
    $ kubectl exec -it $SOURCE_POD -c curl -- curl -sL -o /dev/null -D - http://edition.cnn.com/politics
    HTTP/1.1 301 Moved Permanently
    ...
    location: https://edition.cnn.com/politics
    ...

    command terminated with exit code 35
    {{< /text >}}

    出力に `301 Moved Permanently` が表示される場合は、`ServiceEntry` の設定が正しいことを示します。

1.  `edition.cnn.com` の Egress `Gateway`、ポート 443、および Sidecar リクエストのためのターゲットルールを作成します。Sidecar リクエストは直接 Egress ゲートウェイに向けられます。

{{< tabset category-name="config-api" >}}

{{< tab name="Istio APIs" category-value="istio-apis" >}}

{{< text bash >}}
$ kubectl apply -f - <<EOF
apiVersion: networking.istio.io/v1
kind: Gateway
metadata:
name: istio-egressgateway
spec:
selector:
istio: egressgateway
servers:

- port:
  number: 80
  name: https-port-for-tls-origination
  protocol: HTTPS
  hosts:
  - edition.cnn.com
    tls:
    mode: ISTIO_MUTUAL

---

apiVersion: networking.istio.io/v1
kind: DestinationRule
metadata:
name: egressgateway-for-cnn
spec:
host: istio-egressgateway.istio-system.svc.cluster.local
subsets:

- name: cnn
  trafficPolicy:
  loadBalancer:
  simple: ROUND_ROBIN
  portLevelSettings: - port:
  number: 80
  tls:
  mode: ISTIO_MUTUAL
  sni: edition.cnn.com
  EOF
  {{< /text >}}

{{< /tab >}}

{{< tab name="Gateway API" category-value="gateway-api" >}}

{{< text bash >}}
$ kubectl apply -f - <<EOF
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
name: cnn-egress-gateway
annotations:
networking.istio.io/service-type: ClusterIP
spec:
gatewayClassName: istio
listeners:

- name: https-listener-for-tls-origination
  hostname: edition.cnn.com
  port: 80
  protocol: HTTPS
  tls:
  mode: Terminate
  options:
  gateway.istio.io/tls-terminate-mode: ISTIO_MUTUAL
  allowedRoutes:
  namespaces:
  from: Same

---

apiVersion: networking.istio.io/v1
kind: DestinationRule
metadata:
name: egressgateway-for-cnn
spec:
host: cnn-egress-gateway-istio.default.svc.cluster.local
trafficPolicy:
loadBalancer:
simple: ROUND_ROBIN
portLevelSettings: - port:
number: 80
tls:
mode: ISTIO_MUTUAL
sni: edition.cnn.com
EOF
{{< /text >}}

{{< /tab >}}

{{< /tabset >}}

4.  Egress ゲートウェイを通じてトラフィックをルーティングするルールを構成します：

{{< tabset category-name="config-api" >}}

{{< tab name="Istio APIs" category-value="istio-apis" >}}

{{< text bash >}}
$ kubectl apply -f - <<EOF
apiVersion: networking.istio.io/v1
kind: VirtualService
metadata:
name: direct-cnn-through-egress-gateway
spec:
hosts:

- edition.cnn.com
  gateways:
- istio-egressgateway
- mesh
  http:
- match:
  - gateways:
    - mesh
      port: 80
      route:
  - destination:
    host: istio-egressgateway.istio-system.svc.cluster.local
    subset: cnn
    port:
    number: 80
    weight: 100
- match: - gateways: - istio-egressgateway
  port: 80
  route: - destination:
  host: edition.cnn.com
  port:
  number: 443
  weight: 100
  EOF
  {{< /text >}}

{{< /tab >}}

{{< tab name="Gateway API" category-value="gateway-api" >}}

{{< text bash >}}
$ kubectl apply -f - <<EOF
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
name: direct-cnn-to-egress-gateway
spec:
parentRefs:

- kind: ServiceEntry
  group: networking.istio.io
  name: cnn
  rules:
- backendRefs:
  - name: cnn-egress-gateway-istio
    port: 80

---

apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
name: forward-cnn-from-egress-gateway
spec:
parentRefs:

- name: cnn-egress-gateway
  hostnames:
- edition.cnn.com
  rules:
- backendRefs: - kind: Hostname
  group: networking.istio.io
  name: edition.cnn.com
  port: 443
  EOF
  {{< /text >}}

{{< /tab >}}

{{< /tabset >}}

5.  `edition.cnn.com` のリクエストを実行するための `DestinationRule` を定義します：

    {{< text bash >}}
    $ kubectl apply -f - <<EOF
    apiVersion: networking.istio.io/v1
    kind: DestinationRule
    metadata:
    name: originate-tls-for-edition-cnn-com
    spec:
    host: edition.cnn.com
    trafficPolicy:
    loadBalancer:
    simple: ROUND_ROBIN
    portLevelSettings: - port:
    number: 443
    tls:
    mode: SIMPLE # edition.cnn.com への接続に対して HTTPS を発行
    EOF
    {{< /text >}}

6.  [http://edition.cnn.com/politics](https://edition.cnn.com/politics) への HTTP リクエストを送信します。

    {{< text bash >}}
    $ kubectl exec -it $SOURCE_POD -c curl -- curl -sL -o /dev/null -D - http://edition.cnn.com/politics
    HTTP/1.1 200 OK
    ...
    {{< /text >}}

    出力は、[出口トラフィックの TLS オリジネーション](/ja/docs/tasks/traffic-management/egress/egress-tls-origination/) の例で表示されるものと同じになり、TLS オリジネーション後に _301 Moved Permanently_ メッセージが表示されなくなります。

7.  Egress ゲートウェイプロキシのログを確認します。

{{< tabset category-name="config-api" >}}

{{< tab name="Istio APIs" category-value="istio-apis" >}}

Istio が `istio-system` 名前空間にデプロイされている場合、ログを表示するコマンドは：

{{< text bash >}}
$ kubectl logs -l istio=egressgateway -c istio-proxy -n istio-system | tail
{{< /text >}}

以下のような行が表示されるはずです：

{{< text plain>}}
[2020-06-30T16:17:56.763Z] "GET /politics HTTP/2" 200 - "-" "-" 0 1295938 529 89 "10.244.0.171" "curl/7.64.0" "cf76518d-3209-9ab7-a1d0-e6002728ef5b" "edition.cnn.com" "151.101.129.67:443" outbound|443||edition.cnn.com 10.244.0.170:54280 10.244.0.170:8080 10.244.0.171:35628 - -
{{< /text >}}

{{< /tab >}}

{{< tab name="Gateway API" category-value="gateway-api" >}}

Istio が生成した Pod ラベルを使用して Egress ゲートウェイのログにアクセスします：

{{< text bash >}}
$ kubectl logs -l gateway.networking.k8s.io/gateway-name=cnn-egress-gateway -c istio-proxy | tail
{{< /text >}}

以下のような行が表示されるはずです：

{{< text plain >}}
[2024-03-14T18:37:01.451Z] "GET /politics HTTP/1.1" 200 - via_upstream - "-" 0 2484998 59 37 "172.30.239.26" "curl/7.87.0-DEV" "b80c8732-8b10-4916-9a73-c3e1c848ed1e" "edition.cnn.com" "151.101.131.5:443" outbound|443||edition.cnn.com 172.30.239.33:51270 172.30.239.33:80 172.30.239.26:35192 edition.cnn.com default.forward-cnn-from-egress-gateway.0
{{< /text >}}

{{< /tab >}}

{{< /tabset >}}

### TLS オリジネーションの例をクリーンアップ {#cleanup-the-TLS-origination-example}

作成した Istio 設定項目を削除します：

{{< tabset category-name="config-api" >}}

{{< tab name="Istio APIs" category-value="istio-apis" >}}

{{< text bash >}}
$ kubectl delete gw istio-egressgateway
$ kubectl delete serviceentry cnn
$ kubectl delete virtualservice direct-cnn-through-egress-gateway
$ kubectl delete destinationrule originate-tls-for-edition-cnn-com
$ kubectl delete destinationrule egressgateway-for-cnn
{{< /text >}}

{{< /tab >}}

{{< tab name="Gateway API" category-value="gateway-api" >}}

{{< text bash >}}
$ kubectl delete serviceentry cnn
$ kubectl delete gtw cnn-egress-gateway
$ kubectl delete httproute direct-cnn-to-egress-gateway
$ kubectl delete httproute forward-cnn-from-egress-gateway
$ kubectl delete destinationrule egressgateway-for-cnn
$ kubectl delete destinationrule originate-tls-for-edition-cnn-com
{{< /text >}}

{{< /tab >}}

{{< /tabset >}}

## Egress ゲートウェイで双方向 TLS オリジネーションを行う {#perform-mutual-TLS-origination-with-an-egress-gateway}

前の章と同様に、この章では Egress ゲートウェイを使用して外部サービスへの TLS 接続を発行する方法を説明しますが、今回はサービスが双方向 TLS を要求します。

この例では、以下の手順を実行する必要があります：

1. クライアント証明書とサーバー証明書を生成
1. 双方向 TLS をサポートする外部サービスをデプロイ
1. 必要な証明書を使用して Egress ゲートウェイを再デプロイ

その後、出口トラフィックが Egress ゲートウェイを通過し、Egress ゲートウェイが TLS 接続を発行します。

### クライアント証明書とサーバー証明書を生成 {#generate-client-and-server-certificates-and-keys}

このタスクでは、[openssl](https://man.openbsd.org/openssl.1) を使用して証明書とキーを生成できます。

1. サービスの署名証明書を作成するためのルート証明書と秘密鍵を作成します：

   {{< text bash >}}
    $ openssl req -x509 -sha256 -nodes -days 365 -newkey rsa:2048 -subj '/O=example Inc./CN=example.com' -keyout example.com.key -out example.com.crt
   {{< /text >}}

1. `my-nginx.mesh-external.svc.cluster.local` の証明書と秘密鍵を作成します：

   {{< text bash >}}
    $ openssl req -out my-nginx.mesh-external.svc.cluster.local.csr -newkey rsa:2048 -nodes -keyout my-nginx.mesh-external.svc.cluster.local.key -subj "/CN=my-nginx.mesh-external.svc.cluster.local/O=some organization"
    $ openssl x509 -req -sha256 -days 365 -CA example.com.crt -CAkey example.com.key -set_serial 0 -in my-nginx.mesh-external.svc.cluster.local.csr -out my-nginx.mesh-external.svc.cluster.local.crt
   {{< /text >}}

   または、SAN 検証を有効にしたい場合は、証明書に `SubjectAltNames` を追加できます。例：

   {{< text syntax=bash snip_id=none >}}
    $ cat > san.conf <<EOF
    [req]
    distinguished_name = req_distinguished_name
    req_extensions = v3_req
    x509_extensions = v3_req
    prompt = no
    [req_distinguished_name]
    countryName = US
    [v3_req]
    keyUsage = critical, digitalSignature, keyEncipherment
    extendedKeyUsage = serverAuth, clientAuth
    basicConstraints = critical, CA:FALSE
    subjectAltName = critical, @alt_names
    [alt_names]
    DNS = my-nginx.mesh-external.svc.cluster.local
    EOF
    $
    $ openssl req -out my-nginx.mesh-external.svc.cluster.local.csr -newkey rsa:4096 -nodes -keyout my-nginx.mesh-external.svc.cluster.local.key -subj "/CN=my-nginx.mesh-external.svc.cluster.local/O=some organization" -config san.conf
    $ openssl x509 -req -sha256 -days 365 -CA example.com.crt -CAkey example.com.key -set_serial 0 -in my-nginx.mesh-external.svc.cluster.local.csr -out my-nginx.mesh-external.svc.cluster.local.crt -extfile san.conf -extensions v3_req
   {{< /text >}}

1. クライアント証明書と秘密鍵を生成します：

   {{< text bash >}}
    $ openssl req -out client.example.com.csr -newkey rsa:2048 -nodes -keyout client.example.com.key -subj "/CN=client.example.com/O=client organization"
    $ openssl x509 -req -sha256 -days 365 -CA example.com.crt -CAkey example.com.key -set_serial 1 -in client.example.com.csr -out client.example.com.crt
   {{< /text >}}

### 双方向 TLS サーバーをデプロイ {#deploy-a-mutual-TLS-server}

双方向 TLS プロトコルをサポートする外部サービスを模倣するために、Kubernetes クラスターに [NGINX](https://www.nginx.com) サーバーをデプロイします。
このサーバーは Istio サービスメッシュの外部にあり、例えば、[Istio Sidecar proxy](/ja/docs/setup/additional-setup/sidecar-injection/#deploying-an-app) が有効になっていない名前空間で実行されます。

1.  Istio サービスメッシュの外部を表す名前空間 `mesh-external` を作成します。
    この名前空間では、[Sidecar の自動注入](/ja/docs/setup/additional-setup/sidecar-injection/#automatic-sidecar-injection) は有効になっていません。

    {{< text bash >}}
    $ kubectl create namespace mesh-external
    {{< /text >}}

1.  Kubernetes [Secret](https://kubernetes.io/zh-cn/docs/concepts/configuration/secret/) を作成し、サーバーと CA の証明書を保存します。

    {{< text bash >}}
    $ kubectl create -n mesh-external secret tls nginx-server-certs --key my-nginx.mesh-external.svc.cluster.local.key --cert my-nginx.mesh-external.svc.cluster.local.crt
    $ kubectl create -n mesh-external secret generic nginx-ca-certs --from-file=example.com.crt
    {{< /text >}}

1.  NGINX サーバーの設定ファイルを生成します：

    {{< text bash >}}
    $ cat <<\EOF > ./nginx.conf
    events {
    }

    http {
    log_format main '$remote_addr - $remote_user [$time_local] $status '
      '"$request" $body_bytes_sent "$http_referer" '
    '"$http_user_agent" "$http_x_forwarded_for"';
    access_log /var/log/nginx/access.log main;
    error_log /var/log/nginx/error.log;

    server {
    listen 443 ssl;

        root /usr/share/nginx/html;
        index index.html;

        server_name my-nginx.mesh-external.svc.cluster.local;
        ssl_certificate /etc/nginx-server-certs/tls.crt;
        ssl_certificate_key /etc/nginx-server-certs/tls.key;
        ssl_client_certificate /etc/nginx-ca-certs/example.com.crt;
        ssl_verify_client on;

    }
    }
    EOF
    {{< /text >}}

1.  Kubernetes [ConfigMap](https://kubernetes.io/zh-cn/docs/tasks/configure-pod-container/configure-pod-configmap/) を作成し、NGINX サーバーの設定ファイルを保存します：

    {{< text bash >}}
    $ kubectl create configmap nginx-configmap -n mesh-external --from-file=nginx.conf=./nginx.conf
    {{< /text >}}

1.  NGINX サーバーをデプロイします：

    {{< text bash >}}
    $ kubectl apply -f - <<EOF
    apiVersion: v1
    kind: Service
    metadata:
    name: my-nginx
    namespace: mesh-external
    labels:
    run: my-nginx
    spec:
    ports:

    - port: 443
      protocol: TCP
      selector:
      run: my-nginx

    ***

    apiVersion: apps/v1
    kind: Deployment
    metadata:
    name: my-nginx
    namespace: mesh-external
    spec:
    selector:
    matchLabels:
    run: my-nginx
    replicas: 1
    template:
    metadata:
    labels:
    run: my-nginx
    spec:
    containers: - name: my-nginx
    image: nginx
    ports: - containerPort: 443
    volumeMounts: - name: nginx-config
    mountPath: /etc/nginx
    readOnly: true - name: nginx-server-certs
    mountPath: /etc/nginx-server-certs
    readOnly: true - name: nginx-ca-certs
    mountPath: /etc/nginx-ca-certs
    readOnly: true
    volumes: - name: nginx-config
    configMap:
    name: nginx-configmap - name: nginx-server-certs
    secret:
    secretName: nginx-server-certs - name: nginx-ca-certs
    secret:
    secretName: nginx-ca-certs
    EOF
    {{< /text >}}

1.  `nginx.example.com` の `ServiceEntry` と `VirtualService` を定義し、Istio が `nginx.example.com` のトラフィックを NGINX サーバーに向けるように指示します：

    {{< text bash >}}
    $ kubectl apply -f - <<EOF
    apiVersion: networking.istio.io/v1
    kind: ServiceEntry
    metadata:
    name: nginx
    spec:
    hosts:

    - nginx.example.com
      ports:
    - number: 80
      name: http
      protocol: HTTP
    - number: 443
      name: https
      protocol: HTTPS
      resolution: DNS
      endpoints:
    - address: my-nginx.mesh-external.svc.cluster.local
      ports:
      https: 443

    ***

    apiVersion: networking.istio.io/v1
    kind: VirtualService
    metadata:
    name: nginx
    spec:
    hosts:

    - nginx.example.com
      tls:
    - match: - port: 443
      sni_hosts: - nginx.example.com
      route: - destination:
      host: nginx.example.com
      port:
      number: 443
      weight: 100
      EOF
      {{< /text >}}

### 双方向 TLS オリジネーションのための出口トラフィックを構成 {#configure-mutual-TLS-origination-for-egress-traffic}

1.  部署した Egress ゲートウェイと同じ名前空間に Kubernetes [Secret](https://kubernetes.io/zh-cn/docs/concepts/configuration/secret/) を作成し、クライアントの証明書を保存します：

{{< tabset category-name="config-api" >}}

{{< tab name="Istio APIs" category-value="istio-apis" >}}

{{< text bash >}}
$ kubectl create secret -n istio-system generic client-credential --from-file=tls.key=client.example.com.key \
 --from-file=tls.crt=client.example.com.crt --from-file=ca.crt=example.com.crt
{{< /text >}}

Istio は様々なツールとの統合をサポートするために、異なる Secret 形式をサポートします。
この例では、キー `tls.key`、`tls.crt`、`ca.crt` を持つ一般的な Secret を使用します。

{{< tip >}}
{{< boilerplate crl-tip >}}
{{< /tip >}}

{{< /tab >}}

{{< tab name="Gateway API" category-value="gateway-api" >}}

{{< text bash >}}
$ kubectl create secret -n default generic client-credential --from-file=tls.key=client.example.com.key \
 --from-file=tls.crt=client.example.com.crt --from-file=ca.crt=example.com.crt
{{< /text >}}

Istio は様々なツールとの統合をサポートするために、異なる Secret 形式をサポートします。
この例では、キー `tls.key`、`tls.crt`、`ca.crt` を持つ一般的な Secret を使用します。

{{< tip >}}
{{< boilerplate crl-tip >}}
{{< /tip >}}

{{< /tab >}}

{{< /tabset >}}

2.  `my-nginx.mesh-external.svc.cluster.local`、ポート 443 の Egress `Gateway` を作成し、Egress ゲートウェイに向けられる Sidecar リクエストのためのターゲットルールを作成します：

{{< tabset category-name="config-api" >}}

{{< tab name="Istio APIs" category-value="istio-apis" >}}

{{< text bash >}}
$ kubectl apply -f - <<EOF
apiVersion: networking.istio.io/v1
kind: Gateway
metadata:
name: istio-egressgateway
spec:
selector:
istio: egressgateway
servers:

- port:
  number: 443
  name: https
  protocol: HTTPS
  hosts:
  - my-nginx.mesh-external.svc.cluster.local
    tls:
    mode: ISTIO_MUTUAL

---

apiVersion: networking.istio.io/v1
kind: DestinationRule
metadata:
name: egressgateway-for-nginx
spec:
host: istio-egressgateway.istio-system.svc.cluster.local
subsets:

- name: nginx
  trafficPolicy:
  loadBalancer:
  simple: ROUND_ROBIN
  portLevelSettings: - port:
  number: 443
  tls:
  mode: ISTIO_MUTUAL
  sni: my-nginx.mesh-external.svc.cluster.local
  EOF
  {{< /text >}}

{{< /tab >}}

{{< tab name="Gateway API" category-value="gateway-api" >}}

{{< text bash >}}
$ kubectl apply -f - <<EOF
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
name: nginx-egressgateway
annotations:
networking.istio.io/service-type: ClusterIP
spec:
gatewayClassName: istio
listeners:

- name: https
  hostname: my-nginx.mesh-external.svc.cluster.local
  port: 443
  protocol: HTTPS
  tls:
  mode: Terminate
  options:
  gateway.istio.io/tls-terminate-mode: ISTIO_MUTUAL
  allowedRoutes:
  namespaces:
  from: Same

---

apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
name: nginx-egressgateway-istio-sds
rules:

- apiGroups:
  - ""
    resources:
  - secrets
    verbs:
  - get
  - watch
  - list

---

apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
name: nginx-egressgateway-istio-sds
roleRef:
apiGroup: rbac.authorization.k8s.io
kind: Role
name: nginx-egressgateway-istio-sds
subjects:

- kind: ServiceAccount
  name: nginx-egressgateway-istio

---

apiVersion: networking.istio.io/v1
kind: DestinationRule
metadata:
name: egressgateway-for-nginx
spec:
host: nginx-egressgateway-istio.default.svc.cluster.local
trafficPolicy:
loadBalancer:
simple: ROUND_ROBIN
portLevelSettings: - port:
number: 443
tls:
mode: ISTIO_MUTUAL
sni: my-nginx.mesh-external.svc.cluster.local
EOF
{{< /text >}}

{{< /tab >}}

{{< /tabset >}}

3.  Egress ゲートウェイを通じてトラフィックをルーティングするルールを構成します：

{{< tabset category-name="config-api" >}}

{{< tab name="Istio APIs" category-value="istio-apis" >}}

{{< text bash >}}
$ kubectl apply -f - <<EOF
apiVersion: networking.istio.io/v1
kind: VirtualService
metadata:
name: direct-nginx-through-egress-gateway
spec:
hosts:

- my-nginx.mesh-external.svc.cluster.local
  gateways:
- istio-egressgateway
- mesh
  http:
- match:
  - gateways:
    - mesh
      port: 80
      route:
  - destination:
    host: istio-egressgateway.istio-system.svc.cluster.local
    subset: nginx
    port:
    number: 443
    weight: 100
- match: - gateways: - istio-egressgateway
  port: 443
  route: - destination:
  host: my-nginx.mesh-external.svc.cluster.local
  port:
  number: 443
  weight: 100
  EOF
  {{< /text >}}

{{< /tab >}}

{{< tab name="Gateway API" category-value="gateway-api" >}}

{{< text bash >}}
$ kubectl apply -f - <<EOF
apiVersion: networking.istio.io/v1
kind: VirtualService
metadata:
name: direct-nginx-to-egress-gateway
spec:
hosts:

- my-nginx.mesh-external.svc.cluster.local
  gateways:
- mesh
  http:
- match:
  - port: 80
    route:
  - destination:
    host: nginx-egressgateway-istio.default.svc.cluster.local
    port:
    number: 443

---

apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
name: forward-nginx-from-egress-gateway
spec:
parentRefs:

- name: nginx-egressgateway
  hostnames:
- my-nginx.mesh-external.svc.cluster.local
  rules:
- backendRefs:
  - name: my-nginx
    namespace: mesh-external
    port: 443

---

apiVersion: gateway.networking.k8s.io/v1beta1
kind: ReferenceGrant
metadata:
name: my-nginx-reference-grant
namespace: mesh-external
spec:
from: - group: gateway.networking.k8s.io
kind: HTTPRoute
namespace: default
to: - group: ""
kind: Service
name: my-nginx
EOF
{{< /text >}}

TODO：なぜ `HTTPRoute` が機能しない `VirtualService` の代わりに使用されるかを理解する。
これは完全に `HTTPRoute` を無視し、ターゲットサービスに渡そうとしますが、タイムアウトします。
上記の `VirtualService` との唯一の違いは、生成された `VirtualService` に注釈が付いていることです：`internal.istio.io/route-semantics": "gateway"`。

{{< /tab >}}

{{< /tabset >}}

4.  `DestinationRule` を追加して双方向 TLS オリジネーションを実行します：

{{< tabset category-name="config-api" >}}

{{< tab name="Istio APIs" category-value="istio-apis" >}}

{{< text bash >}}
$ kubectl apply -n istio-system -f - <<EOF
apiVersion: networking.istio.io/v1
kind: DestinationRule
metadata:
name: originate-mtls-for-nginx
spec:
host: my-nginx.mesh-external.svc.cluster.local
trafficPolicy:
loadBalancer:
simple: ROUND_ROBIN
portLevelSettings: - port:
number: 443
tls:
mode: MUTUAL
credentialName: client-credential # これは、クライアント証明書を保存した Secret と一致する必要があります
sni: my-nginx.mesh-external.svc.cluster.local # subjectAltNames: # 証明書が前のセクションで指定された SAN によって生成された場合は、有効にできます # - my-nginx.mesh-external.svc.cluster.local
EOF
{{< /text >}}

{{< /tab >}}

{{< tab name="Gateway API" category-value="gateway-api" >}}

{{< text bash >}}
$ kubectl apply -f - <<EOF
apiVersion: networking.istio.io/v1
kind: DestinationRule
metadata:
name: originate-mtls-for-nginx
spec:
host: my-nginx.mesh-external.svc.cluster.local
trafficPolicy:
loadBalancer:
simple: ROUND_ROBIN
portLevelSettings: - port:
number: 443
tls:
mode: MUTUAL
credentialName: client-credential # これは、クライアント証明書を保存した Secret と一致する必要があります
sni: my-nginx.mesh-external.svc.cluster.local # subjectAltNames: # 証明書が前のセクションで指定された SAN によって生成された場合は、有効にできます # - my-nginx.mesh-external.svc.cluster.local
EOF
{{< /text >}}

{{< /tab >}}

{{< /tabset >}}

{{< boilerplate auto-san-validation >}}

5.  証明書と秘密鍵が Egress ゲートウェイに提供され、アクティブな状態であることを確認します：

{{< tabset category-name="config-api" >}}

{{< tab name="Istio APIs" category-value="istio-apis" >}}

{{< text bash >}}
$ istioctl -n istio-system proxy-config secret deploy/istio-egressgateway | grep client-credential
kubernetes://client-credential Cert Chain ACTIVE true 1 2024-06-04T12:46:28Z 2023-06-05T12:46:28Z
kubernetes://client-credential-cacert Cert Chain ACTIVE true 16491643791048004260 2024-06-04T12:46:28Z 2023-06-05T12:46:28Z
{{< /text >}}

{{< /tab >}}

{{< tab name="Gateway API" category-value="gateway-api" >}}

{{< text bash >}}
$ istioctl proxy-config secret deploy/nginx-egressgateway-istio | grep client-credential
kubernetes://client-credential Cert Chain ACTIVE true 1 2024-06-04T12:46:28Z 2023-06-05T12:46:28Z
kubernetes://client-credential-cacert Cert Chain ACTIVE true 16491643791048004260 2024-06-04T12:46:28Z 2023-06-05T12:46:28Z
{{< /text >}}

{{< /tab >}}

{{< /tabset >}}

6.  `http://my-nginx.mesh-external.svc.cluster.local` への HTTP リクエストを送信します：

    {{< text bash >}}
    $ kubectl exec "$(kubectl get pod -l app=curl -o jsonpath={.items..metadata.name})" -c curl -- curl -sS http://my-nginx.mesh-external.svc.cluster.local
    <!DOCTYPE html>
    <html>
    <head>
    <title>Welcome to nginx!</title>
    ...
    {{< /text >}}

7.  Egress ゲートウェイプロキシのログを確認します：

{{< tabset category-name="config-api" >}}

{{< tab name="Istio APIs" category-value="istio-apis" >}}

Istio が `istio-system` 名前空間にデプロイされている場合、ログを表示するコマンドは：

{{< text bash >}}
$ kubectl logs -l istio=egressgateway -n istio-system | grep 'my-nginx.mesh-external.svc.cluster.local' | grep HTTP
{{< /text >}}

以下のような行が表示されるはずです：

{{< text plain>}}
[2018-08-19T18:20:40.096Z] "GET / HTTP/1.1" 200 - 0 612 7 5 "172.30.146.114" "curl/7.35.0" "b942b587-fac2-9756-8ec6-303561356204" "my-nginx.mesh-external.svc.cluster.local" "172.21.72.197:443"
{{< /text >}}

{{< /tab >}}

{{< tab name="Gateway API" category-value="gateway-api" >}}

Istio が生成した Pod ラベルを使用して Egress ゲートウェイのログにアクセスします：

{{< text bash >}}
$ kubectl logs -l gateway.networking.k8s.io/gateway-name=nginx-egressgateway | grep 'my-nginx.mesh-external.svc.cluster.local' | grep HTTP
{{< /text >}}

以下のような行が表示されるはずです：

{{< text plain >}}
[2024-04-08T20:08:18.451Z] "GET / HTTP/1.1" 200 - via_upstream - "-" 0 615 5 5 "172.30.239.41" "curl/7.87.0-DEV" "86e54df0-6dc3-46b3-a8b8-139474c32a4d" "my-nginx.mesh-external.svc.cluster.local" "172.30.239.57:443" outbound|443||my-nginx.mesh-external.svc.cluster.local 172.30.239.53:48530 172.30.239.53:443 172.30.239.41:53694 my-nginx.mesh-external.svc.cluster.local default.forward-nginx-from-egress-gateway.0
{{< /text >}}

{{< /tab >}}

{{< /tabset >}}

### 双方向 TLS オリジネーションの例をクリーンアップ {#cleanup-the-mutual-TLS-origination-example}

1.  NGINX 双方向 TLS サーバーリソースを削除します：

    {{< text bash >}}
    $ kubectl delete secret nginx-server-certs nginx-ca-certs -n mesh-external
    $ kubectl delete configmap nginx-configmap -n mesh-external
    $ kubectl delete service my-nginx -n mesh-external
    $ kubectl delete deployment my-nginx -n mesh-external
    $ kubectl delete namespace mesh-external
    {{< /text >}}

1.  ゲートウェイ設定リソースを削除します：

{{< tabset category-name="config-api" >}}

{{< tab name="Istio APIs" category-value="istio-apis" >}}

{{< text bash >}}
$ kubectl delete secret client-credential -n istio-system
$ kubectl delete gw istio-egressgateway
$ kubectl delete virtualservice direct-nginx-through-egress-gateway
$ kubectl delete destinationrule -n istio-system originate-mtls-for-nginx
$ kubectl delete destinationrule egressgateway-for-nginx
{{< /text >}}

{{< /tab >}}

{{< tab name="Gateway API" category-value="gateway-api" >}}

{{< text bash >}}
$ kubectl delete secret client-credential
$ kubectl delete gtw nginx-egressgateway
$ kubectl delete role nginx-egressgateway-istio-sds
$ kubectl delete rolebinding nginx-egressgateway-istio-sds
$ kubectl delete virtualservice direct-nginx-to-egress-gateway
$ kubectl delete httproute forward-nginx-from-egress-gateway
$ kubectl delete destinationrule originate-mtls-for-nginx
$ kubectl delete destinationrule egressgateway-for-nginx
$ kubectl delete referencegrant my-nginx-reference-grant -n mesh-external
{{< /text >}}

{{< /tab >}}

{{< /tabset >}}

3.  証明書と秘密鍵を削除します：

    {{< text bash >}}
    $ rm example.com.crt example.com.key my-nginx.mesh-external.svc.cluster.local.crt my-nginx.mesh-external.svc.cluster.local.key my-nginx.mesh-external.svc.cluster.local.csr client.example.com.crt client.example.com.csr client.example.com.key
    {{< /text >}}

4.  生成された設定ファイルを削除します

    {{< text bash >}}
    $ rm ./nginx.conf
    $ rm ./gateway-patch.json
    {{< /text >}}

## クリーンアップ {#cleanup}

`curl` の Service と Deployment を削除します：

{{< text bash >}}
$ kubectl delete -f @samples/curl/curl.yaml@
{{< /text >}}
