---
title: cert-manager
description: cert-manager との統合方法に関する説明。
weight: 26
keywords: [integration, cert-manager]
aliases:
  - /zh/docs/tasks/traffic-management/ingress/ingress-certmgr/
  - /zh/docs/examples/advanced-gateways/ingress-certmgr/
owner: istio/wg-environments-maintainers
test: no
---

[cert-manager](https://cert-manager.io/) は証明書管理を自動化するツールで、Istio Gateway と統合して TLS 証明書を管理できます。

## 設定 {#configuration}

[cert-manager のインストールドキュメント](https://cert-manager.io/docs/installation/kubernetes/)を参照してクイックスタートできます。特別な設定なしで Istio と一緒に使用できます。

## 利用方法 {#usage}

### Istio Gateway {#istio-gateway}

cert-manager は Kubernetes に Secret キーを書き込み、Gateway からその Secret を参照できます。

1. まず、[cert-manager の Issuer ドキュメント](https://cert-manager.io/docs/configuration/)に従って `Issuer` リソースを設定します。`Issuer` は証明書認証局（CA）を表す Kubernetes リソースで、証明書署名リクエストに応じて署名証明書を発行できます。例：

   {{< text yaml >}}
   apiVersion: cert-manager.io/v1
   kind: Issuer
   metadata:
   name: ca-issuer
   namespace: istio-system
   spec:
   ca:
   secretName: ca-key-pair
   {{< /text >}}

   {{< tip >}}
   一般的な発行者タイプである ACME では、Pod とサービスを作成してチャレンジリクエストに応答し、クライアントがドメインを所有していることを検証します。これらのチャレンジに対応するには、`http://<YOUR_DOMAIN>/.well-known/acme-challenge/<TOKEN>` にアクセスできる必要があります。設定は実装ごとに異なる場合があります。
   {{< /tip >}}

1. 次に、[cert-manager ドキュメント](https://cert-manager.io/docs/usage/certificate/)に従って `Certificate` リソースを設定します。`Certificate` は `istio-ingressgateway` デプロイと同じ名前空間で作成する必要があります。例：

   {{< text yaml >}}
   apiVersion: cert-manager.io/v1
   kind: Certificate
   metadata:
   name: ingress-cert
   namespace: istio-system
   spec:
   secretName: ingress-cert
   commonName: my.example.com
   dnsNames:

   - my.example.com
     ...
     {{< /text >}}

1. `Certificate` リソースを作成すると、`istio-system` 名前空間に Secret が作成されます。これを Gateway の `tls` 設定の `credentialName` フィールドで参照できます：

   {{< text yaml >}}
   apiVersion: networking.istio.io/v1
   kind: Gateway
   metadata:
   name: gateway
   spec:
   selector:
   istio: ingressgateway
   servers:

   - port:
     number: 443
     name: https
     protocol: HTTPS
     tls:
     mode: SIMPLE
     credentialName: ingress-cert # これは Certificate の secretName と一致させる必要があります
     hosts: - my.example.com # これは Certificate の DNS 名と一致させる必要があります
     {{< /text >}}

### Kubernetes Ingress {#kubernetes-ingress}

cert-manager は [Ingress オブジェクトへのアノテーション設定](https://cert-manager.io/docs/usage/ingress/)により、Kubernetes Ingress と直接統合できます。この方法を使う場合、Ingress は `istio-ingressgateway` Deployment と同じ名前空間である必要があります。Secret は同じ名前空間内でのみ参照可能だからです。

または、[Istio Gateway](#istio-gateway) の手順で `Certificate` を作成し、`Ingress` オブジェクトで参照することもできます：

{{< text yaml >}}
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
name: ingress
annotations:
kubernetes.io/ingress.class: istio
spec:
rules:

- host: my.example.com
  http: ...
  tls:
- hosts: - my.example.com # これは証明書の DNS 名と一致させる必要があります
  secretName: ingress-cert # これは証明書の Secret 名と一致させる必要があります
  {{< /text >}}
