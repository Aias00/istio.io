---
title: Ingress Sidecar TLS 終端
description: Ingress Gateway を使用せず、Sidecar 上で TLS トラフィックを終端する方法を説明します。
weight: 30
keywords: [traffic-management,ingress,https]
owner: istio/wg-networking-maintainers
test: yes
---

通常の Istio メッシュデプロイメントでは、下流リクエストの TLS 終端は Ingress Gateway で行われます。
これは多くのユースケースで十分ですが、メッシュ内の API ゲートウェイなど一部のケースでは Ingress Gateway が不要な場合もあります。
このタスクでは、Istio Ingress Gateway による余分なホップを排除し、アプリケーションとともに動作する Envoy Sidecar がサービスメッシュ外部からのリクエストに対して TLS 終端を行う方法を紹介します。

このタスクで使用するサンプル HTTPS サービスは、シンプルな [httpbin](https://httpbin.org/) サービスです。
以下の手順で、サービスメッシュ内に httpbin サービスをデプロイし、設定します。

{{< boilerplate experimental-feature-warning >}}

## 準備 {#before-you-begin}

- [インストールガイド](/ja/docs/setup/) の手順に従い、実験的機能 `ENABLE_TLS_ON_SIDECAR_INGRESS` を有効にして Istio をセットアップします。

  {{< text bash >}}
  $ istioctl install --set profile=default --set values.pilot.env.ENABLE_TLS_ON_SIDECAR_INGRESS=true
  {{< /text >}}

- test 名前空間を作成し、ターゲットとなる `httpbin` サービスをデプロイします。この名前空間で Sidecar インジェクションを有効にしてください。

  {{< text bash >}}
  $ kubectl create ns test
  $ kubectl label namespace test istio-injection=enabled
  {{< /text >}}

## グローバル mTLS の有効化 {#enable-global-mtls}

以下の `PeerAuthentication` ポリシーを適用し、メッシュ内のすべてのワークロードで mTLS トラフィックを有効にします。

{{< text bash >}}
$ kubectl -n test apply -f - <<EOF
apiVersion: security.istio.io/v1
kind: PeerAuthentication
metadata:
name: default
spec:
mtls:
mode: STRICT
EOF
{{< /text >}}

## 外部公開 httpbin ポートでの PeerAuthentication 無効化 {#disable-peerauthentication-for-the-externally-exposed-httpbin-port}

httpbin Service のポートで `PeerAuthentication` を無効化し、Sidecar で入口 TLS 終端を行います。ここで対象となるのは、外部通信専用の httpbin Service の `targetPort` です。

{{< text bash >}}
$ kubectl -n test apply -f - <<EOF
apiVersion: security.istio.io/v1
kind: PeerAuthentication
metadata:
name: disable-peer-auth-for-external-mtls-port
namespace: test
spec:
selector:
matchLabels:
app: httpbin
mtls:
mode: STRICT
portLevelMtls:
9080:
mode: DISABLE
EOF
{{< /text >}}

## CA 証明書、サーバー証明書/鍵、クライアント証明書/鍵の生成 {#generate-ca-cert-server-certkey-and-client-certkey}

このタスクでは、お好みのツールで証明書と鍵を生成できます。以下のコマンドは [openssl](https://man.openbsd.org/openssl.1) を使用しています：

{{< text bash >}}
$ #CA は example.com
$ openssl req -x509 -sha256 -nodes -days 365 -newkey rsa:2048 -subj '/O=example Inc./CN=example.com' -keyout example.com.key -out example.com.crt
$ #サーバーは httpbin.test.svc.cluster.local
$ openssl req -out httpbin.test.svc.cluster.local.csr -newkey rsa:2048 -nodes -keyout httpbin.test.svc.cluster.local.key -subj "/CN=httpbin.test.svc.cluster.local/O=httpbin organization"
$ openssl x509 -req -days 365 -CA example.com.crt -CAkey example.com.key -set_serial 1 -in httpbin.test.svc.cluster.local.csr -out httpbin.test.svc.cluster.local.crt
$ #クライアントは client.test.svc.cluster.local
$ openssl req -out client.test.svc.cluster.local.csr -newkey rsa:2048 -nodes -keyout client.test.svc.cluster.local.key -subj "/CN=client.test.svc.cluster.local/O=client organization"
$ openssl x509 -req -days 365 -CA example.com.crt -CAkey example.com.key -set_serial 1 -in client.test.svc.cluster.local.csr -out client.test.svc.cluster.local.crt
{{< /text >}}

## 証明書と鍵の Kubernetes Secret 作成 {#create-k8s-secrets-for-the-certificates-and-keys}

{{< text bash >}}
$ kubectl -n test create secret generic httpbin-mtls-termination-cacert --from-file=ca.crt=./example.com.crt
$ kubectl -n test create secret tls httpbin-mtls-termination --cert ./httpbin.test.svc.cluster.local.crt --key ./httpbin.test.svc.cluster.local.key
{{< /text >}}

## テストサービス httpbin のデプロイ {#deploy-the-httpbin-test-service}

httpbin Deployment を作成する際、`userVolumeMount` アノテーションを使って istio-proxy Sidecar に証明書をマウントします。この手順が必要なのは、現時点で Istio Sidecar が `credentialName` 設定をサポートしていないためです。

{{< text yaml >}}
sidecar.istio.io/userVolume: '{"tls-secret":{"secret":{"secretName":"httpbin-mtls-termination","optional":true}},"tls-ca-secret":{"secret":{"secretName":"httpbin-mtls-termination-cacert"}}}'
sidecar.istio.io/userVolumeMount: '{"tls-secret":{"mountPath":"/etc/istio/tls-certs/","readOnly":true},"tls-ca-secret":{"mountPath":"/etc/istio/tls-ca-certs/","readOnly":true}}'
{{< /text >}}

以下のコマンドで `userVolumeMount` 設定付きの `httpbin` サービスをデプロイします：

{{< text bash >}}
$ kubectl -n test apply -f - <<EOF
apiVersion: v1
kind: ServiceAccount
metadata:
name: httpbin

---

apiVersion: v1
kind: Service
metadata:
name: httpbin
labels:
app: httpbin
service: httpbin
spec:
ports:

- port: 8443
  name: https
  targetPort: 9080
- port: 8080
  name: http
  targetPort: 9081
  selector:
  app: httpbin

---

apiVersion: apps/v1
kind: Deployment
metadata:
name: httpbin
spec:
replicas: 1
selector:
matchLabels:
app: httpbin
version: v1
template:
metadata:
labels:
app: httpbin
version: v1
annotations:
sidecar.istio.io/userVolume: '{"tls-secret":{"secret":{"secretName":"httpbin-mtls-termination","optional":true}},"tls-ca-secret":{"secret":{"secretName":"httpbin-mtls-termination-cacert"}}}'
sidecar.istio.io/userVolumeMount: '{"tls-secret":{"mountPath":"/etc/istio/tls-certs/","readOnly":true},"tls-ca-secret":{"mountPath":"/etc/istio/tls-ca-certs/","readOnly":true}}'
spec:
serviceAccountName: httpbin
containers: - image: docker.io/kennethreitz/httpbin
imagePullPolicy: IfNotPresent
name: httpbin
ports: - containerPort: 80
EOF
{{< /text >}}

## httpbin の外部 mTLS 有効化設定 {#configure-httpbin-to-enable-external-mtls}

この機能のコアとなるステップです。`Sidecar` API で入口 TLS 設定を構成します。TLS モードは `SIMPLE` または `MUTUAL` で、本例では `MUTUAL` を使用します。

{{< text bash >}}
$ kubectl -n test apply -f - <<EOF
apiVersion: networking.istio.io/v1
kind: Sidecar
metadata:
name: ingress-sidecar
namespace: test
spec:
workloadSelector:
labels:
app: httpbin
version: v1
ingress:

- port:
  number: 9080
  protocol: HTTPS
  name: external
  defaultEndpoint: 0.0.0.0:80
  tls:
  mode: MUTUAL
  privateKey: "/etc/istio/tls-certs/tls.key"
  serverCertificate: "/etc/istio/tls-certs/tls.crt"
  caCertificates: "/etc/istio/tls-ca-certs/ca.crt"
- port:
  number: 9081
  protocol: HTTP
  name: internal
  defaultEndpoint: 0.0.0.0:80
  EOF
  {{< /text >}}

## 検証 {#verification}

httpbin サーバーのデプロイと設定が完了したので、メッシュ内外のエンドツーエンド接続をテストするために 2 つのクライアントを起動します：

1. httpbin サービスと同じ test 名前空間内の内部クライアント（curl、Sidecar 注入済み）
1. default 名前空間（サービスメッシュ外部）の外部クライアント（curl）

{{< text bash >}}
$ kubectl apply -f samples/curl/curl.yaml
$ kubectl -n test apply -f samples/curl/curl.yaml
{{< /text >}}

以下のコマンドで、すべてが起動し、設定が正しいことを確認します。

{{< text bash >}}
$ kubectl get pods
NAME READY STATUS RESTARTS AGE
curl-557747455f-xx88g 1/1 Running 0 4m14s
{{< /text >}}

{{< text bash >}}
$ kubectl get pods -n test
NAME READY STATUS RESTARTS AGE
httpbin-5bbdbd6588-z9vbs 2/2 Running 0 8m44s
curl-557747455f-brzf6 2/2 Running 0 6m57s
{{< /text >}}

{{< text bash >}}
$ kubectl get svc -n test
NAME TYPE CLUSTER-IP EXTERNAL-IP PORT(S) AGE
httpbin ClusterIP 10.100.78.113 <none> 8443/TCP,8080/TCP 10m
curl ClusterIP 10.110.35.153 <none> 80/TCP 8m49s
{{< /text >}}

以下のコマンドで `httpbin-5bbdbd6588-z9vbs` を httpbin Pod 名に置き換えてください。

{{< text bash >}}
$ istioctl proxy-config secret httpbin-5bbdbd6588-z9vbs.test
RESOURCE NAME TYPE STATUS VALID CERT SERIAL NUMBER NOT AFTER NOT BEFORE
file-cert:/etc/istio/tls-certs/tls.crt~/etc/istio/tls-certs/tls.key Cert Chain ACTIVE true 1 2023-02-14T09:51:56Z 2022-02-14T09:51:56Z
default Cert Chain ACTIVE true 329492464719328863283539045344215802956 2022-02-15T09:55:46Z 2022-02-14T09:53:46Z
ROOTCA CA ACTIVE true 204427760222438623495455009380743891800 2032-02-07T16:58:00Z 2022-02-09T16:58:00Z
file-root:/etc/istio/tls-ca-certs/ca.crt Cert Chain ACTIVE true 14033888812979945197 2023-02-14T09:51:56Z 2022-02-14T09:51:56Z
{{< /text >}}

### ポート 8080 でのメッシュ内部接続の検証 {#verify-internal-mesh-connectivity-on-port-8080}

{{< text bash >}}
$ export INTERNAL_CLIENT=$(kubectl -n test get pod -l app=curl -o jsonpath={.items..metadata.name})
$ kubectl -n test exec "${INTERNAL_CLIENT}" -c curl -- curl -IsS "http://httpbin:8080/status/200"
HTTP/1.1 200 OK
server: envoy
date: Mon, 24 Oct 2022 09:04:52 GMT
content-type: text/html; charset=utf-8
access-control-allow-origin: \*
access-control-allow-credentials: true
content-length: 0
x-envoy-upstream-service-time: 5
{{< /text >}}

### ポート 8443 での外部からメッシュ内部への接続検証 {#verify-external-to-internal-mesh-connectivity-on-port-8443}

外部クライアントからの mTLS トラフィックを検証するには、まず CA 証明書とクライアント証明書/鍵を default 名前空間で動作する curl クライアントにコピーします。

{{< text bash >}}
$ export EXTERNAL_CLIENT=$(kubectl get pod -l app=curl -o jsonpath={.items..metadata.name})
$ kubectl cp client.test.svc.cluster.local.key default/"${EXTERNAL_CLIENT}":/tmp/
$ kubectl cp client.test.svc.cluster.local.crt default/"${EXTERNAL_CLIENT}":/tmp/
$ kubectl cp example.com.crt default/"${EXTERNAL_CLIENT}":/tmp/ca.crt
{{< /text >}}

証明書が外部 curl クライアントで利用可能になったら、以下のコマンドで内部 httpbin サービスへの接続を検証できます。

{{< text bash >}}
$ kubectl exec "${EXTERNAL_CLIENT}" -c curl -- curl -IsS --cacert /tmp/ca.crt --key /tmp/client.test.svc.cluster.local.key --cert /tmp/client.test.svc.cluster.local.crt -HHost:httpbin.test.svc.cluster.local "https://httpbin.test.svc.cluster.local:8443/status/200"
server: istio-envoy
date: Mon, 24 Oct 2022 09:05:31 GMT
content-type: text/html; charset=utf-8
access-control-allow-origin: _
access-control-allow-credentials: true
content-length: 0
x-envoy-upstream-service-time: 4
x-envoy-decorator-operation: ingress-sidecar.test:9080/_
{{< /text >}}

入口ポート 8443 で外部 mTLS 接続を検証するだけでなく、ポート 8080 で外部 mTLS トラフィックが受け付けられないことも確認してください。

{{< text bash >}}
$ kubectl exec "${EXTERNAL_CLIENT}" -c curl -- curl -IsS --cacert /tmp/ca.crt --key /tmp/client.test.svc.cluster.local.key --cert /tmp/client.test.svc.cluster.local.crt -HHost:httpbin.test.svc.cluster.local "http://httpbin.test.svc.cluster.local:8080/status/200"
curl: (56) Recv failure: Connection reset by peer
command terminated with exit code 56
{{< /text >}}

## 双方向 TLS 終端サンプルのクリーンアップ {#cleanup-the-mutual-tls-termination-example}

1.  作成した Kubernetes リソースを削除します：

    {{< text bash >}}
    $ kubectl delete secret httpbin-mtls-termination httpbin-mtls-termination-cacert -n test
    $ kubectl delete service httpbin curl -n test
    $ kubectl delete deployment httpbin curl -n test
    $ kubectl delete namespace test
    $ kubectl delete service curl
    $ kubectl delete deployment curl
    {{< /text >}}

1.  証明書と秘密鍵を削除します：

    {{< text bash >}}
    $ rm example.com.crt example.com.key httpbin.test.svc.cluster.local.crt httpbin.test.svc.cluster.local.key httpbin.test.svc.cluster.local.csr \
     client.test.svc.cluster.local.crt client.test.svc.cluster.local.key client.test.svc.cluster.local.csr
    {{< /text >}}

1.  Istio をクラスターからアンインストールします：

    {{< text bash >}}
    $ istioctl uninstall --purge -y
    {{< /text >}}
