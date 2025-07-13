---
title: TLS 終端なしの Ingress Gateway
description: Ingress Gateway で SNI パススルーを構成する方法。
weight: 30
keywords: [traffic-management, ingress, https]
aliases:
  - /zh/docs/examples/advanced-gateways/ingress-sni-passthrough/
owner: istio/wg-networking-maintainers
test: yes
---

[セキュアゲートウェイ](/ja/docs/tasks/traffic-management/ingress/secure-ingress/) では、HTTP サービスの HTTPS アクセスエントリの構成方法を説明しています。本例では、HTTPS サービスの HTTPS アクセスエントリの構成、すなわち Ingress Gateway で TLS 終端を行わず SNI パススルーを構成する方法を説明します。

このタスクの HTTPS サンプルサービスはシンプルな [NGINX](https://www.nginx.com) サービスです。以下の手順で、まず Kubernetes クラスタに NGINX サービスを作成し、次にこのサービスに `nginx.example.com` というドメイン名でアクセスできるようにゲートウェイを構成します。

{{< boilerplate gateway-api-gamma-experimental >}}

## 準備 {#before-you-begin}

[インストールガイド](/ja/docs/setup/) に従って Istio をデプロイしてください。

## クライアントおよびサーバー証明書と鍵の生成 {#generate-client-and-server-certificates-and-keys}

このタスクでは、お好みのツールで証明書と鍵を生成できます。以下のコマンドは [openssl](https://man.openbsd.org/openssl.1) を使用しています：

1. サービス証明書の署名用にルート証明書と秘密鍵を作成します：

   {{< text bash >}}
    $ openssl req -x509 -sha256 -nodes -days 365 -newkey rsa:2048 -subj '/O=example Inc./CN=example.com' -keyout example.com.key -out example.com.crt
   {{< /text >}}

1. `nginx.example.com` 用の証明書と秘密鍵を作成します：

   {{< text bash >}}
    $ openssl req -out nginx.example.com.csr -newkey rsa:2048 -nodes -keyout nginx.example.com.key -subj "/CN=nginx.example.com/O=some organization"
    $ openssl x509 -req -sha256 -days 365 -CA example.com.crt -CAkey example.com.key -set_serial 0 -in nginx.example.com.csr -out nginx.example.com.crt
   {{< /text >}}

## NGINX サービスのデプロイ {#deploy-an-nginx-server}

1.  サービスの証明書を保存する Kubernetes の [Secret](https://kubernetes.io/ja/docs/concepts/configuration/secret/) リソースを作成します：

    {{< text bash >}}
    $ kubectl create secret tls nginx-server-certs --key nginx.example.com.key --cert nginx.example.com.crt
    {{< /text >}}

1.  NGINX サービス用の設定ファイルを作成します：

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

        server_name nginx.example.com;
        ssl_certificate /etc/nginx-server-certs/tls.crt;
        ssl_certificate_key /etc/nginx-server-certs/tls.key;

    }
    }
    EOF
    {{< /text >}}

1.  NGINX サービスの設定を保存する Kubernetes の [ConfigMap](https://kubernetes.io/ja/docs/tasks/configure-pod-container/configure-pod-configmap/) リソースを作成します：

    {{< text bash >}}
    $ kubectl create configmap nginx-configmap --from-file=nginx.conf=./nginx.conf
    {{< /text >}}

1.  NGINX サービスをデプロイします：

    {{< text bash >}}
    $ cat <<EOF | istioctl kube-inject -f - | kubectl apply -f -
    apiVersion: v1
    kind: Service
    metadata:
    name: my-nginx
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
    readOnly: true
    volumes: - name: nginx-config
    configMap:
    name: nginx-configmap - name: nginx-server-certs
    secret:
    secretName: nginx-server-certs
    EOF
    {{< /text >}} 

1.  NGINX サービスが正しくデプロイされたかをテストするには、Sidecar プロキシからリクエストを送り、サーバ証明書の検証をスキップします（`curl` の `-k` オプションを使用）。サーバ証明書の `common name (CN)` が `nginx.example.com` であることを確認してください。

    {{< text bash >}}
    $ kubectl exec "$(kubectl get pod -l run=my-nginx -o jsonpath={.items..metadata.name})" -c istio-proxy -- curl -sS -v -k --resolve nginx.example. com:443:127.0.0.1 https://nginx.example.com
    ...
    SSL connection using TLSv1.2 / ECDHE-RSA-AES256-GCM-SHA384
    ALPN, server accepted to use http/1.1
    Server certificate:
    subject: CN=nginx.example.com; O=some organization
    start date: May 27 14:18:47 2020 GMT
    expire date: May 27 14:18:47 2021 GMT
    issuer: O=example Inc.; CN=example.com
    SSL certificate verify result: unable to get local issuer certificate (20), continuing anyway.
 
    > GET / HTTP/1.1
    > User-Agent: curl/7.58.0
    > Host: nginx.example.com
    > ...
    > < HTTP/1.1 200 OK
 
    < Server: nginx/1.17.10
    ...
    <!DOCTYPE html>
    <html>
    <head>
    <title>Welcome to nginx!</title>
    ...
    {{< /text >}}

## Ingress Gateway の構成 {#configure-an-ingress-gateway}

1. ポート 443 の `server` セクションを持つ `Gateway` を定義します。`PASSTHROUGH tls` TLS モードに注意してください。このモードは Gateway が入口トラフィックをそのまま（AS IS）パススルーし、TLS 終端を行わないことを示します。

   {{< text bash >}}
    $ kubectl apply -f - <<EOF
    apiVersion: networking.istio.io/v1
    kind: Gateway
    metadata:
    name: mygateway
    spec:
    selector:
    istio: ingressgateway # istio デフォルトの入口ゲートウェイを使用
    servers:

    - port:
      number: 443
      name: https
      protocol: HTTPS
      tls:
      mode: PASSTHROUGH
      hosts: - nginx.example.com
      EOF
      {{< /text >}}

1. `Gateway` 経由で入ってくるトラフィックのルーティングを構成します：

   {{< text bash >}}
    $ kubectl apply -f - <<EOF
    apiVersion: networking.istio.io/v1
    kind: VirtualService
    metadata:
    name: nginx
    spec:
    hosts:

    - nginx.example.com
      gateways:
    - mygateway
      tls:
    - match: - port: 443
      sniHosts: - nginx.example.com
      route: - destination:
      host: my-nginx
      port:
      number: 443
      EOF
      {{< /text >}}

1. [Ingress IP とポートの確認](/ja/docs/tasks/traffic-management/ingress/ingress-control/#determining-the-ingress-i-p-and-ports) の指示に従い、`SECURE_INGRESS_PORT` と `INGRESS_HOST` の環境変数を定義します。

1. クラスタ外から NGINX サービスにアクセスします。サーバ証明書が正しく返され、証明書が正常に検証されたこと（**SSL certificate verify ok** の出力）が確認できます。

   {{< text bash >}}
    $ curl -v --resolve "nginx.example.com:$SECURE_INGRESS_PORT:$INGRESS_HOST" --cacert example.com.crt "https://nginx.example.com:$SECURE_INGRESS_PORT"
    Server certificate:
    subject: CN=nginx.example.com; O=some organization
    start date: Wed, 15 Aug 2018 07:29:07 GMT
    expire date: Sun, 25 Aug 2019 07:29:07 GMT
    issuer: O=example Inc.; CN=example.com
    SSL certificate verify ok.

    < HTTP/1.1 200 OK
    < Server: nginx/1.15.2
    ...
      <html>
      <head>
      <title>Welcome to nginx!</title>
    {{< /text >}}

## クリーンアップ {#cleanup}

1. 作成した Kubernetes リソースを削除します：

   {{< text bash >}}
    $ kubectl delete secret nginx-server-certs
    $ kubectl delete configmap nginx-configmap
    $ kubectl delete service my-nginx
    $ kubectl delete deployment my-nginx
    $ kubectl delete gateway mygateway
    $ kubectl delete virtualservice nginx
   {{< /text >}}

1. 証明書と鍵を削除します：

   {{< text bash >}}
    $ rm example.com.crt example.com.key nginx.example.com.crt nginx.example.com.key nginx.example.com.csr
   {{< /text >}}

1. 本サンプルで生成した設定ファイルを削除します：

   {{< text bash >}}
    $ rm ./nginx.conf
   {{< /text >}}
