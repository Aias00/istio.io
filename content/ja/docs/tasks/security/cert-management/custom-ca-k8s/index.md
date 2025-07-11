---
title: Kubernetes CSR を使ったカスタム CA 統合
description: カスタム証明書認証局（Kubernetes CSR API と統合）を使って Istio ワークロード証明書を提供する方法を紹介します。
weight: 100
keywords: [security, certificate]
aliases:
  - /zh/docs/tasks/security/custom-ca-k8s/
owner: istio/wg-security-maintainers
test: no
status: Experimental
---

{{< boilerplate experimental >}}

この機能には Kubernetes バージョン >= 1.18 が必要です。

このタスクでは、[Kubernetes CSR API](https://kubernetes.io/zh-cn/docs/reference/access-authn-authz/certificate-signing-requests/) と統合されたカスタム証明書認証局を使ってワークロード証明書を発行する方法を紹介します。異なるワークロードは異なる証明書署名者によって署名された証明書を取得でき、各証明書署名者は本質的に異なる CA です。ワークロードの証明書が同じ証明書署名者によって発行されていれば、これらのワークロードは MTLS 通信が可能ですが、異なる署名者によるワークロード間では通信できません。この機能は [Chiron](/ja/blog/2019/dns-cert/) を利用します。Chiron は Istiod に関連付けられた軽量コンポーネントで、Kubernetes CSR API を使って証明書に署名します。

この例では、[オープンソースの cert-manager](https://cert-manager.io) を使用します。cert-manager はバージョン 1.4 以降で [Kubernetes `CertificateSigningRequests` の実験的サポート](https://cert-manager.io/docs/usage/kube-csr/) を追加しています。

## Kubernetes クラスターにカスタム CA コントローラーをデプロイする {#deploy-custom-ca-controller-in-the-k8s-cluster}

1. [インストールドキュメント](https://cert-manager.io/docs/installation/) に従って cert-manager をデプロイします。

   {{< warning >}}
   `--feature-gates=ExperimentalCertificateSigningRequestControllers=true` フィーチャーゲートが有効になっていることを確認してください。
   {{< /warning >}}

   {{< text bash >}}
   $ helm repo add jetstack https://charts.jetstack.io
   $ helm repo update
   $ helm install cert-manager jetstack/cert-manager --namespace cert-manager --create-namespace --set featureGates="ExperimentalCertificateSigningRequestControllers=true" --set installCRDs=true
   {{< /text >}}

1. cert-manager 用に 3 つの自己署名クラスタ発行者：`istio-system`、`foo`、`bar` を作成します。
   ※名前空間発行者や他のタイプの発行者も利用可能です。

   {{< text bash >}}
   $ cat <<EOF > ./selfsigned-issuer.yaml
   apiVersion: cert-manager.io/v1
   kind: ClusterIssuer
   metadata:
   name: selfsigned-bar-issuer
   spec:
   selfSigned: {}

   ***

   apiVersion: cert-manager.io/v1
   kind: Certificate
   metadata:
   name: bar-ca
   namespace: cert-manager
   spec:
   isCA: true
   commonName: bar
   secretName: bar-ca-selfsigned
   issuerRef:
   name: selfsigned-bar-issuer
   kind: ClusterIssuer
   group: cert-manager.io

   ***

   apiVersion: cert-manager.io/v1
   kind: ClusterIssuer
   metadata:
   name: bar
   spec:
   ca:
   secretName: bar-ca-selfsigned

   ***

   apiVersion: cert-manager.io/v1
   kind: ClusterIssuer
   metadata:
   name: selfsigned-foo-issuer
   spec:
   selfSigned: {}

   ***

   apiVersion: cert-manager.io/v1
   kind: Certificate
   metadata:
   name: foo-ca
   namespace: cert-manager
   spec:
   isCA: true
   commonName: foo
   secretName: foo-ca-selfsigned
   issuerRef:
   name: selfsigned-foo-issuer
   kind: ClusterIssuer
   group: cert-manager.io

   ***

   apiVersion: cert-manager.io/v1
   kind: ClusterIssuer
   metadata:
   name: foo
   spec:
   ca:
   secretName: foo-ca-selfsigned

   ***

   apiVersion: cert-manager.io/v1
   kind: ClusterIssuer
   metadata:
   name: selfsigned-istio-issuer
   spec:
   selfSigned: {}

   ***

   apiVersion: cert-manager.io/v1
   kind: Certificate
   metadata:
   name: istio-ca
   namespace: cert-manager
   spec:
   isCA: true
   commonName: istio-system
   secretName: istio-ca-selfsigned
   issuerRef:
   name: selfsigned-istio-issuer
   kind: ClusterIssuer
   group: cert-manager.io

   ***

   apiVersion: cert-manager.io/v1
   kind: ClusterIssuer
   metadata:
   name: istio-system
   spec:
   ca:
   secretName: istio-ca-selfsigned
   EOF
   $ kubectl apply -f ./selfsigned-issuer.yaml
   {{< /text >}}

## 各クラスタ発行者のシークレットを確認する {#verify-secrets-are-created-for-each-cluster-issuer}

{{< text bash >}}
$ kubectl get secret -n cert-manager -l controller.cert-manager.io/fao=true
NAME TYPE DATA AGE
bar-ca-selfsigned kubernetes.io/tls 3 3m36s
foo-ca-selfsigned kubernetes.io/tls 3 3m36s
istio-ca-selfsigned kubernetes.io/tls 3 3m38s
{{< /text >}}

## 各クラスタ発行者のルート証明書をエクスポートする {#export-root-certificates-for-each-cluster-issuer}

{{< text bash >}}
$ export ISTIOCA=$(kubectl get clusterissuers istio-system -o jsonpath='{.spec.ca.secretName}' | xargs kubectl get secret -n cert-manager -o jsonpath='{.data.ca\.crt}' | base64 -d | sed 's/^/        /')
$ export FOOCA=$(kubectl get clusterissuers foo -o jsonpath='{.spec.ca.secretName}' | xargs kubectl get secret -n cert-manager -o jsonpath='{.data.ca\.crt}' | base64 -d | sed 's/^/        /')
$ export BARCA=$(kubectl get clusterissuers bar -o jsonpath='{.spec.ca.secretName}' | xargs kubectl get secret -n cert-manager -o jsonpath='{.data.ca\.crt}' | base64 -d | sed 's/^/ /')
{{< /text >}}

## デフォルトの証明書署名者情報で Istio をデプロイする {#deploy-istio-with-default-cert-signer-info}

1. 以下の設定で `istioctl` を使って Istio をデプロイします。`ISTIO_META_CERT_SIGNER` はワークロードが使うデフォルトの証明書署名者です。

   {{< text bash >}}
   $ cat <<EOF > ./istio.yaml
   apiVersion: install.istio.io/v1alpha1
   kind: IstioOperator
   spec:
   values:
   pilot:
   env:
   EXTERNAL_CA: ISTIOD_RA_KUBERNETES_API
   meshConfig:
   defaultConfig:
   proxyMetadata:
   ISTIO_META_CERT_SIGNER: istio-system
   caCertificates: - pem: |
   $ISTIOCA
          certSigners:
          - clusterissuers.cert-manager.io/istio-system
        - pem: |
    $FOOCA
   certSigners: - clusterissuers.cert-manager.io/foo - pem: |
   $BARCA
          certSigners:
          - clusterissuers.cert-manager.io/bar
      components:
        pilot:
          k8s:
            env:
            - name: CERT_SIGNER_DOMAIN
              value: clusterissuers.cert-manager.io
            - name: PILOT_CERT_PROVIDER
              value: k8s.io/clusterissuers.cert-manager.io/istio-system
            overlays:
              - kind: ClusterRole
                name: istiod-clusterrole-istio-system
                patches:
                  - path: rules[-1]
                    value: |
                      apiGroups:
                      - certificates.k8s.io
                      resourceNames:
                      - clusterissuers.cert-manager.io/foo
                      - clusterissuers.cert-manager.io/bar
                      - clusterissuers.cert-manager.io/istio-system
                      resources:
                      - signers
                      verbs:
                      - approve
    EOF
    $ istioctl install --skip-confirmation -f ./istio.yaml
   {{< /text >}}

1. `bar` と `foo` 名前空間を作成します。

   {{< text bash >}}
   $ kubectl create ns bar
   $ kubectl create ns foo
   {{< /text >}}

1. `bar` 名前空間で `proxyconfig-bar.yaml` をデプロイし、`bar` 名前空間のワークロード用の証明書署名者を定義します。

   {{< text bash >}}
   $ cat <<EOF > ./proxyconfig-bar.yaml
   apiVersion: networking.istio.io/v1beta1
   kind: ProxyConfig
   metadata:
   name: barpc
   namespace: bar
   spec:
   environmentVariables:
   ISTIO_META_CERT_SIGNER: bar
   EOF
   $ kubectl apply -f ./proxyconfig-bar.yaml
   {{< /text >}}

1. `foo` 名前空間で `proxyconfig-foo.yaml` をデプロイし、`foo` 名前空間のワークロード用の証明書署名者を定義します。

   {{< text bash >}}
   $ cat <<EOF > ./proxyconfig-foo.yaml
   apiVersion: networking.istio.io/v1beta1
   kind: ProxyConfig
   metadata:
   name: foopc
   namespace: foo
   spec:
   environmentVariables:
   ISTIO_META_CERT_SIGNER: foo
   EOF
   $ kubectl apply -f ./proxyconfig-foo.yaml
   {{< /text >}}

1. `foo` と `bar` 名前空間に `httpbin` と `curl` サンプルアプリケーションをデプロイします。

   {{< text bash >}}
   $ kubectl label ns foo istio-injection=enabled
   $ kubectl label ns bar istio-injection=enabled
   $ kubectl apply -f samples/httpbin/httpbin.yaml -n foo
   $ kubectl apply -f samples/curl/curl.yaml -n foo
   $ kubectl apply -f samples/httpbin/httpbin.yaml -n bar
   {{< /text >}}

## 同じ名前空間内の `httpbin` と `curl` 間のネットワーク接続を検証する {#verify-network-connectivity-between-httpbin-and-curl-within-a-namespace}

ワークロードをデプロイすると、それぞれ関連する署名者情報を持つ CSR リクエストが送信されます。Istiod はこれらの CSR リクエストをカスタム CA に転送して署名します。カスタム CA は正しいクラスタ発行者を使って証明書に署名します。`foo` 名前空間のワークロードは `foo` クラスタ発行者を、`bar` 名前空間のワークロードは `bar` クラスタ発行者を使います。正しいクラスタ発行者で署名されていることを検証するには、同じ名前空間内のワークロード同士は通信でき、異なる名前空間間では通信できないことを確認します。

1.  `CURL_POD_FOO` 環境変数に `curl` Pod の名前を設定します。

    {{< text bash >}}
    $ export CURL_POD_FOO=$(kubectl get pod -n foo -l app=curl -o jsonpath={.items..metadata.name})
    {{< /text >}}

1.  `foo` 名前空間の `curl` と `httpbin` サービス間のネットワーク接続を確認します。

    {{< text bash >}}
    $ kubectl exec "$CURL_POD_FOO" -n foo -c curl -- curl http://httpbin.foo:8000/html
    <!DOCTYPE html>
    <html>
      <head>
      </head>
      <body>
          <h1>Herman Melville - Moby-Dick</h1>

          <div>
            <p>
              Availing himself of the mild...
            </p>
          </div>

      </body>
     {{< /text >}}

1.  `foo` 名前空間の `curl` サービスと `bar` 名前空間の `httpbin` サービス間のネットワーク接続を確認します。

    {{< text bash >}}
    $ kubectl exec "$CURL_POD_FOO" -n foo -c curl -- curl http://httpbin.bar:8000/html
    upstream connect error or disconnect/reset before headers. reset reason: connection failure, transport failure reason: TLS error: 268435581:SSL routines:OPENSSL_internal:CERTIFICATE_VERIFY_FAILED
    {{< /text >}}

## クリーンアップ {#cleanup}

- `istio-system`、`foo`、`bar` 名前空間を削除します：

  {{< text bash >}}
  $ kubectl delete ns foo
  $ kubectl delete ns bar
  $ istioctl uninstall --purge -y
  $ helm delete -n cert-manager cert-manager
  $ kubectl delete ns istio-system cert-manager
  $ unset ISTIOCA FOOCA BARCA
  $ rm -rf istio.yaml proxyconfig-foo.yaml proxyconfig-bar.yaml selfsigned-issuer.yaml
  {{< /text >}}

## この機能を使う理由 {#reasons-to-use-this-feature}

- カスタム CA 統合 - Kubernetes CSR リクエストで署名者名を指定することで、Istio が Kubernetes CSR API を通じてカスタム証明書認証局と統合できます。カスタム CA は Kubernetes コントローラーとして `CertificateSigningRequest` や `Certificate` リソースを監視し、アクションを取る必要があります。

- より良いマルチテナンシー - 異なるワークロードごとに異なる証明書署名者を指定することで、異なるテナントのワークロード証明書を異なる CA で署名できます。
