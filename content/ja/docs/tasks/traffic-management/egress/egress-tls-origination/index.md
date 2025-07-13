---
title: Egress TLS オリジネーション
description: Istio で外部サービスへのトラフィックに対して TLS オリジネーションを行う方法を説明します。
keywords: [traffic-management, egress]
weight: 20
aliases:
  - /zh/docs/examples/advanced-gateways/egress-tls-origination/
owner: istio/wg-networking-maintainers
test: yes
---

[出口トラフィックの制御](/ja/docs/tasks/traffic-management/egress/) のタスクでは、サービスメッシュ内のアプリケーションが外部（サービスメッシュ外）の HTTP および HTTPS サービスにアクセスする方法を紹介しました。
そのタスクで説明したように、[`ServiceEntry`](/ja/docs/reference/config/networking/service-entry/) を使って Istio で外部サービスへのアクセスを制御できます。
この例では、Istio の設定で外部サービスへのトラフィックに {{< gloss >}}TLS オリジネーション{{< /gloss >}} を行う方法を説明します。
元のトラフィックが HTTP の場合、Istio はそれを HTTPS 接続に変換します。

## ユースケース {#use-case}

従来のアプリケーションが HTTP で外部サービスと通信しているとします。
そのアプリケーションを運用する組織に「すべての外部トラフィックを暗号化する」新たな要件が課された場合、Istio を使えばアプリケーションのコードを変更せずにこの要件を満たせます。
アプリケーションは暗号化されていない HTTP リクエストを送信し、Istio がリクエストを暗号化します。

アプリケーションから未暗号化の HTTP リクエストを発行し、Istio で TLS アップグレードを行うもう一つの利点は、より良いテレメトリや未暗号化リクエストへの柔軟なルーティング制御が可能になることです。

## 始める前に {#before-you-begin}

- [インストールガイド](/ja/docs/setup/) の手順に従って Istio をデプロイしてください。

- [curl]({{< github_tree >}}/samples/curl) サンプルアプリを起動し、外部呼び出しのテストソースとします。

  [Sidecar の自動注入](/ja/docs/setup/additional-setup/sidecar-injection/#automatic-sidecar-injection)が有効な場合は、次のコマンドで `curl` アプリをデプロイします：

  {{< text bash >}}
    $ kubectl apply -f @samples/curl/curl.yaml@
  {{< /text >}}

  そうでない場合は、`curl` アプリをデプロイする前に手動で Sidecar を注入してください：

  {{< text bash >}}
    $ kubectl apply -f <(istioctl kube-inject -f @samples/curl/curl.yaml@)
  {{< /text >}}

  実際には、`exec` や `curl` が実行できる任意の Pod でこのタスクを行えます。

- 外部サービスへのリクエスト送信に使う Pod 名を保存する環境変数を作成します。
  [curl]({{< github_tree >}}/samples/curl) サンプルを使う場合は：

  {{< text bash >}}
    $ export SOURCE_POD=$(kubectl get pod -l app=curl -o jsonpath={.items..metadata.name})
  {{< /text >}}

## 外部サービスへのアクセスを構成する {#configuring-access-to-an-external-service}

まず、[外部サービスへのアクセス](/ja/docs/tasks/traffic-management/egress/egress-control) タスクと同じ設定で、外部サービス `edition.cnn.com` へのアクセスを構成します。
今回は HTTP/HTTPS 両方のアクセスを単一の `ServiceEntry` で有効化します。

1. `edition.cnn.com` へのアクセスを有効にする `ServiceEntry` を作成します：

   {{< text syntax=bash snip_id=apply_simple >}}
    $ kubectl apply -f - <<EOF
    apiVersion: networking.istio.io/v1
    kind: ServiceEntry
    metadata:
    name: edition-cnn-com
    spec:
    hosts:

    - edition.cnn.com
      ports:
    - number: 80
      name: http-port
      protocol: HTTP
    - number: 443
      name: https-port
      protocol: HTTPS
      resolution: DNS
      EOF
     {{< /text >}}

1. 外部 HTTP サービスにリクエストを送信します：

   {{< text syntax=bash snip_id=curl_simple >}}
    $ kubectl exec "${SOURCE_POD}" -c curl -- curl -sSL -o /dev/null -D - http://edition.cnncom/politics
    HTTP/1.1 301 Moved Permanently
    ...
    location: https://edition.cnn.com/politics
    ...
  
    HTTP/2 200
    ...
   {{< /text >}}

   出力は上記のようになります（一部省略）。

**curl** の `-L` フラグはリダイレクトを追従することを意味します。
この場合、サーバーは `http://edition.cnn.com/politics` への HTTP リクエストにリダイレクト（[301 Moved Permanently](https://tools.ietf.org/html/rfc2616#section-10.3.2)）で応答します。
リダイレクト先は HTTPS の `https://edition.cnn.com/politics` です。
2 回目のリクエストではサーバーはリクエスト内容と **200 OK** を返します。

**curl** コマンドはリダイレクトを簡単に処理しますが、ここには 2 つの問題があります。
1 つ目はリクエストの冗長性で、`http://edition.cnn.com/politics` の内容取得に遅延が 2 倍になります。
2 つ目は URL のパス（この例では **politics**）が平文で送信されることです。
通信を盗聴された場合、アプリがどのページを取得したかが分かってしまいます。
プライバシー保護の観点から、これを防ぎたい場合があります。

Istio で `TLS` オリジネーションを構成すれば、これらの問題を解決できます。

## 出口トラフィックの TLS オリジネーション {#TLS-origination-for-egress-traffic}

1. 前節の `ServiceEntry` と `VirtualService` を再定義し、HTTP リクエストのポートを書き換え、TLS オリジネーションを行う `DestinationRule` を追加します。

   {{< text syntax=bash snip_id=apply_origination >}}
    $ kubectl apply -f - <<EOF
    apiVersion: networking.istio.io/v1
    kind: ServiceEntry
    metadata:
    name: edition-cnn-com
    spec:
    hosts:
  
    - edition.cnn.com
      ports:
    - number: 80
      name: http-port
      protocol: HTTP
      targetPort: 443
    - number: 443
      name: https-port
      protocol: HTTPS
      resolution: DNS
  
    ***
  
    apiVersion: networking.istio.io/v1
    kind: DestinationRule
    metadata:
    name: edition-cnn-com
    spec:
    host: edition.cnn.com
    trafficPolicy:
    portLevelSettings: - port:
    number: 80
    tls:
    mode: SIMPLE # edition.cnn.com へのアクセス時に HTTPS を発行
    EOF
   {{< /text >}}

   上記の `DestinationRule` は、ポート 80 および `ServiceEntry` 上の HTTP リクエストに対して TLS オリジネーションを行います。
   その後、ポート 80 のリクエストはターゲットポート 443 にリダイレクトされます。

1. 前節と同様に `http://edition.cnn.com/politics` へ HTTP リクエストを送信します：

   {{< text syntax=bash snip_id=curl_origination_http >}}
    $ kubectl exec "${SOURCE_POD}" -c curl -- curl -sSL -o /dev/null -D - http://edition.cnn.com/politics
    HTTP/1.1 200 OK
    ...
   {{< /text >}}

   今回は **200 OK** のみが返ります。
   Istio が **curl** のために TLS オリジネーションを行い、元の HTTP が HTTPS にアップグレードされて `edition.cnn.com` に転送されます。
   サーバーはリダイレクトなしで直接内容を返します。これによりクライアントとサーバー間のリクエスト冗長性が解消され、メッシュ内通信が暗号化され、アプリが `edition.cnn.com` の **politics** を取得した事実が秘匿されます。

   なお、前節と同じコマンドを使っています。
   外部サービスにプログラム的にアクセスするアプリケーションでも、コードを変更せずに Istio の設定だけで TLS オリジネーションの恩恵を受けられます。

1. アプリケーションが HTTPS で外部サービスにアクセスしている場合も、従来通り動作します：

   {{< text syntax=bash snip_id=curl_origination_https >}}
    $ kubectl exec "${SOURCE_POD}" -c curl -- curl -sSL -o /dev/null -D - https://edition.cnn.com/politics
    HTTP/2 200
    ...
   {{< /text >}}

## その他のセキュリティ考慮事項 {#additional-security-considerations}

アプリケーション Pod とローカルホスト上の Sidecar プロキシ間のトラフィックは暗号化されていません。
そのため、アプリケーションノードに侵入した攻撃者は、そのノードのローカルネットワーク上の平文通信を傍受できます。
一部の環境では、ノードのローカルネットワーク上も含めてすべてのトラフィックを暗号化する厳格なセキュリティ要件がある場合があります。
このような場合、アプリケーションは必ず HTTPS（TLS）を使うべきであり、本例の TLS オリジネーションだけでは不十分です。

また、アプリケーションが HTTPS リクエストを発行しても、攻撃者は
[サーバー名表示（SNI）](https://ja.wikipedia.org/wiki/Server_Name_Indication) を調べることでクライアントが `edition.cnn.com` にリクエストしていることを知る可能性があります。**SNI** フィールドは TLS ハンドシェイク時に平文で送信されます。
HTTPS を使えばクライアントがどのページにアクセスしたかは秘匿できますが、どのサイト（`edition.cnn.com`）にアクセスしたかまでは秘匿できません。

### TLS オリジネーション設定のクリーンアップ {#cleanup-the-tls-origination-configuration}

作成した Istio 設定を削除します：

{{< text bash >}}
$ kubectl delete serviceentry edition-cnn-com
$ kubectl delete destinationrule edition-cnn-com
{{< /text >}}

## 出口トラフィックの双方向 TLS オリジネーション {#mutual-tls-origination-for-egress-traffic}

このセクションでは、Sidecar で外部サービスへの TLS オリジネーションを構成します。今回は双方向 TLS を要求するサービスを使います。
この例は多くの手順を含み、まず以下の準備が必要です：

1. クライアント証明書とサーバー証明書の生成
1. 双方向 TLS をサポートする外部サービスのデプロイ
1. クライアント（curl Pod）を手順 1 で作成した証明書で構成

これらの準備ができたら、外部トラフィックを Sidecar 経由で TLS オリジネーションできます。

### クライアント証明書・サーバー証明書・クライアント鍵・サーバー鍵の生成 {#generate-client-and-server-certificates-and-keys}

このタスクでは、お好みのツールで証明書と鍵を生成できます。
以下は [openssl](https://man.openbsd.org/openssl.1) を使った例です。

1. サービス用の署名証明書を作成するためのルート証明書と秘密鍵を作成します：

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

### 双方向 TLS サーバーのデプロイ {#deploy-a-mutual-tls-server}

双方向 TLS プロトコルをサポートする外部サービスを模倣するため、Kubernetes クラスタに [NGINX](https://www.nginx.com) サーバーをデプロイします。
このサーバーは Istio サービスメッシュ外部（Sidecar 自動注入が有効でない名前空間）で動作します。

1.  Istio サービスメッシュ外部を表す名前空間 `mesh-external` を作成します。
    この名前空間では Sidecar 自動注入は有効になっていません。

    {{< text bash >}}
    $ kubectl create namespace mesh-external
    {{< /text >}}

1.  Kubernetes [Secret](https://kubernetes.io/ja/docs/concepts/configuration/secret/) を作成し、サーバー証明書と CA 証明書を保存します。

    {{< text bash >}}
    $ kubectl create -n mesh-external secret tls nginx-server-certs --key my-nginx.mesh-external.svc.cluster.local.key --cert my-nginx.mesh-external.svc.cluster.local.crt
    $ kubectl create -n mesh-external secret generic nginx-ca-certs --from-file=example.com.crt
    {{< /text >}}

1.  NGINX サーバーの設定ファイルを作成します：

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

1.  Kubernetes [ConfigMap](https://kubernetes.io/ja/docs/tasks/configure-pod-container/configure-pod-configmap/) を作成し、NGINX サーバーの設定ファイルを保存します：

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
    annotations:
    "networking.istio.io/exportTo": "." # 外部サービスをシミュレートするため、この名前空間外にはエクスポートしない
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

### クライアント（curl Pod）の構成 {#configure-the-client-curl-pod}

1. Kubernetes [Secret](https://kubernetes.io/ja/docs/concepts/configuration/secret/) を作成し、クライアント証明書を保存します：

   {{< text bash >}}
    $ kubectl create secret generic client-credential --from-file=tls.key=client.example.com.key \
    --from-file=tls.crt=client.example.com.crt --from-file=ca.crt=example.com.crt
   {{< /text >}}

    **必ず**クライアント Pod をデプロイする名前空間（本例では `default`）で作成してください。

   {{< boilerplate crl-tip >}}

1. 上記で作成した Secret へのアクセス権をクライアント Pod（ここでは `curl`）に与えるため、必要な `RBAC` を作成します：

   {{< text bash >}}
    $ kubectl create role client-credential-role --resource=secret --verb=list
    $ kubectl create rolebinding client-credential-role-binding --role=client-credential-role --serviceaccount=default:curl
   {{< /text >}}

### Sidecar で出口トラフィックの双方向 TLS オリジネーションを構成 {#configure-mutual-tls-origination-for-egress-traffic-at-sidecar}

1. `ServiceEntry` を追加して HTTP リクエストを 443 ポートにリダイレクトし、双方向 TLS オリジネーションを行う `DestinationRule` を追加します：

   {{< text bash >}}
    $ kubectl apply -f - <<EOF
    apiVersion: networking.istio.io/v1
    kind: ServiceEntry
    metadata:
    name: originate-mtls-for-nginx
    spec:
    hosts:

    - my-nginx.mesh-external.svc.cluster.local
      ports:
    - number: 80
      name: http-port
      protocol: HTTP
      targetPort: 443
    - number: 443
      name: https-port
      protocol: HTTPS
      resolution: DNS

    ***

    apiVersion: networking.istio.io/v1
    kind: DestinationRule
    metadata:
    name: originate-mtls-for-nginx
    spec:
    workloadSelector:
    matchLabels:
    app: curl
    host: my-nginx.mesh-external.svc.cluster.local
    trafficPolicy:
    loadBalancer:
    simple: ROUND_ROBIN
    portLevelSettings: - port:
    number: 80
    tls:
    mode: MUTUAL
    credentialName: client-credential # これはクライアント証明書を保存した Secret と一致し、DR に  workloadSelector がある場合のみ有効
    sni: my-nginx.mesh-external.svc.cluster.local # subjectAltNames: # 証明書が SAN で生成されている場 合は有効化可能（前節参照） # - my-nginx.mesh-external.svc.cluster.local
    EOF
   {{< /text >}}

    上記の `DestinationRule` は 80 ポートの HTTP に対して mTLS オリジネーションを行い、`ServiceEntry` で 80 ポートのリクエストを 443 ポートにリダイレクトします。

   {{< boilerplate auto-san-validation >}}

1. 証明書が Sidecar に提供され、アクティブであることを確認します：

   {{< text bash >}}
    $ istioctl proxy-config secret deploy/curl | grep client-credential
    kubernetes://client-credential Cert Chain ACTIVE true 1 2024-06-04T12:15:20Z  2023-06-05T12:15:20Z
    kubernetes://client-credential-cacert Cert Chain ACTIVE true 10792363984292733914  2024-06-04T12:15:19Z 2023-06-05T12:15:19Z
   {{< /text >}}

1. `http://my-nginx.mesh-external.svc.cluster.local` へ HTTP リクエストを送信します：

   {{< text bash >}}
    $ kubectl exec "$(kubectl get pod -l app=curl -o jsonpath={.items..metadata.name})" -c curl -- curl -sS http://my-nginx.mesh-external.svc.cluster.local
    <!DOCTYPE html>
    <html>
    <head>
    <title>Welcome to nginx!</title>
    ...
   {{< /text >}}

1. `curl` Pod のログにリクエストに対応する行があるか確認します：

   {{< text bash >}}
    $ kubectl logs -l app=curl -c istio-proxy | grep 'my-nginx.mesh-external.svc.cluster.local'
   {{< /text >}}

   以下のような出力が表示されるはずです：

   {{< text plain>}}
    [2022-05-19T10:01:06.795Z] "GET / HTTP/1.1" 200 - via_upstream - "-" 0 615 1 0 "-" "curl/7.83.1-DEV" "96e8d8a7-92ce-9939-aa47-9f5f530a69fb" "my-nginx.mesh-external.svc.cluster.local:443" "10.107.176.65:443"
   {{< /text >}}

### 双方向 TLS オリジネーション設定のクリーンアップ {#cleanup-the-mutual-tls-origination-configuration}

1. 作成した Kubernetes リソースを削除します：

   {{< text bash >}}
    $ kubectl delete secret nginx-server-certs nginx-ca-certs -n mesh-external
    $ kubectl delete secret client-credential
    $ kubectl delete rolebinding client-credential-role-binding
    $ kubectl delete role client-credential-role
    $ kubectl delete configmap nginx-configmap -n mesh-external
    $ kubectl delete service my-nginx -n mesh-external
    $ kubectl delete deployment my-nginx -n mesh-external
    $ kubectl delete namespace mesh-external
    $ kubectl delete serviceentry originate-mtls-for-nginx
    $ kubectl delete destinationrule originate-mtls-for-nginx
   {{< /text >}}

1. 証明書と秘密鍵を削除します：

   {{< text bash >}}
    $ rm example.com.crt example.com.key my-nginx.mesh-external.svc.cluster.local.crt my-nginx.mesh-external.svc.cluster.local.key my-nginx.mesh-external.svc.cluster.local.csr client.example.com.crt client.example.com.csr client.example.com.key
   {{< /text >}}

1. この例で使った・生成した設定ファイルを削除します：

   {{< text bash >}}
    $ rm ./nginx.conf
   {{< /text >}}

## 共通設定のクリーンアップ {#cleanup-common-configuration}

`curl` Service と Deployment を削除します：

{{< text bash >}}
$ kubectl delete service curl
$ kubectl delete deployment curl
{{< /text >}}
