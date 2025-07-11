---
title: リモートからのテレメトリアドオンアクセス
description: このタスクでは、Istio テレメトリアドオンセットへの外部アクセスを設定する方法を紹介します。
weight: 98
keywords:
  [telemetry, gateway, jaeger, zipkin, tracing, kiali, prometheus, addons]
aliases:
  - /zh/docs/tasks/telemetry/gateways/
owner: istio/wg-policies-and-telemetry-maintainers
test: yes
---

このタスクでは、Istio を設定してクラスタ外部からテレメトリアアドオンにアクセス・公開する方法を説明します。

## リモートアクセスの設定 {#configuring-remote-access}

テレメトリアアドオンへのリモートアクセスにはさまざまな方法があります。
このタスクでは、基本的な 2 つのアクセス方法（セキュア（HTTPS）と非セキュア（HTTP））を扱います。
本番や機密性の高い環境では、**必ず**セキュアな方法を利用してください。
非セキュアアクセスは設定が簡単ですが、クラスタ外に送信される認証情報やデータは保護されません。

どちらの方法でも、まず以下の手順を実行してください：

1. クラスタに[Istio をインストール](/zh/docs/setup/install/istioctl)します。

   追加のテレメトリアドオンをインストールするには、[統合](/zh/docs/ops/integrations/)ドキュメントを参照してください。

1. これらのアドオンを公開するためのドメイン名を設定します。この例では、各アドオンを `grafana.example.com` のようなサブドメインで公開します。

   - ドメイン名（例：example.com）が `istio-ingressgateway` の外部 IP に向いている場合：

     {{< text bash >}}
     $ export INGRESS_DOMAIN="example.com"
     {{< /text >}}

   - ドメイン名がない場合は [`nip.io`](https://nip.io/) を利用できます。これは指定した IP アドレスに自動で解決されますが、本番用途には推奨されません。

     {{< text bash >}}
     $ export INGRESS_HOST=$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
        $ export INGRESS_DOMAIN=${INGRESS_HOST}.nip.io
     {{< /text >}}

### 方法 1：セキュアアクセス（HTTPS）{#option-one-secure-access-HTTPS}

セキュアアクセスにはサーバ証明書が必要です。以下の手順でドメイン用の証明書をインストール・設定します。

{{< warning >}}
この方法は**転送層のセキュリティのみ**をカバーします。アドオンを外部公開する場合は、認証も必ず設定してください。
{{< /warning >}}

この例では自己署名証明書を使いますが、本番用途には[cert-manager](/zh/docs/ops/integrations/certmanager/)などの利用を検討してください。
HTTPS ゲートウェイの基本については[HTTPS でゲートウェイを保護](/zh/docs/tasks/traffic-management/ingress/secure-ingress/)も参照してください。

1. 証明書を設定します（この例では `openssl` で自己署名証明書を作成）：

   {{< text bash >}}
   $ CERT_DIR=/tmp/certs
   $ mkdir -p ${CERT_DIR}
    $ openssl req -x509 -sha256 -nodes -days 365 -newkey rsa:2048 -subj "/O=example Inc./CN=_.${INGRESS_DOMAIN}" -keyout ${CERT_DIR}/ca.key -out ${CERT_DIR}/ca.crt
    $ openssl req -out ${CERT_DIR}/cert.csr -newkey rsa:2048 -nodes -keyout ${CERT_DIR}/tls.key -subj "/CN=_.${INGRESS_DOMAIN}/O=example organization"
    $ openssl x509 -req -sha256 -days 365 -CA ${CERT_DIR}/ca.crt -CAkey ${CERT_DIR}/ca.key -set_serial 0 -in ${CERT_DIR}/cert.csr -out ${CERT_DIR}/tls.crt
    $ kubectl create -n istio-system secret tls telemetry-gw-cert --key=${CERT_DIR}/tls.key --cert=${CERT_DIR}/tls.crt
   {{< /text >}}

1. テレメトリアドオンのネットワーク設定を適用します。

   1. Grafana を公開する設定：

      {{< text bash >}}
      $ cat <<EOF | kubectl apply -f -
      apiVersion: networking.istio.io/v1
      kind: Gateway
      metadata:
      name: grafana-gateway
      namespace: istio-system
      spec:
      selector:
      istio: ingressgateway
      servers:

      - port:
        number: 443
        name: https-grafana
        protocol: HTTPS
        tls:
        mode: SIMPLE
        credentialName: telemetry-gw-cert
        hosts:
        - "grafana.${INGRESS_DOMAIN}"

      ***

      apiVersion: networking.istio.io/v1
      kind: VirtualService
      metadata:
      name: grafana-vs
      namespace: istio-system
      spec:
      hosts:

      - "grafana.${INGRESS_DOMAIN}"
        gateways:
      - grafana-gateway
        http:
      - route:
        - destination:
          host: grafana
          port:
          number: 3000

      ***

      apiVersion: networking.istio.io/v1
      kind: DestinationRule
      metadata:
      name: grafana
      namespace: istio-system
      spec:
      host: grafana
      trafficPolicy:
      tls:
      mode: DISABLE

      ***

      EOF
      gateway.networking.istio.io/grafana-gateway created
      virtualservice.networking.istio.io/grafana-vs created
      destinationrule.networking.istio.io/grafana created
      {{< /text >}}

   1. アプリケーション Kiali を公開する設定：

      {{< text bash >}}
      $ cat <<EOF | kubectl apply -f -
      apiVersion: networking.istio.io/v1
      kind: Gateway
      metadata:
      name: kiali-gateway
      namespace: istio-system
      spec:
      selector:
      istio: ingressgateway
      servers:

      - port:
        number: 443
        name: https-kiali
        protocol: HTTPS
        tls:
        mode: SIMPLE
        credentialName: telemetry-gw-cert
        hosts:
        - "kiali.${INGRESS_DOMAIN}"

      ***

      apiVersion: networking.istio.io/v1
      kind: VirtualService
      metadata:
      name: kiali-vs
      namespace: istio-system
      spec:
      hosts:

      - "kiali.${INGRESS_DOMAIN}"
        gateways:
      - kiali-gateway
        http:
      - route:
        - destination:
          host: kiali
          port:
          number: 20001

      ***

      apiVersion: networking.istio.io/v1
      kind: DestinationRule
      metadata:
      name: kiali
      namespace: istio-system
      spec:
      host: kiali
      trafficPolicy:
      tls:
      mode: DISABLE

      ***

      EOF
      gateway.networking.istio.io/kiali-gateway created
      virtualservice.networking.istio.io/kiali-vs created
      destinationrule.networking.istio.io/kiali created
      {{< /text >}}

   1. Prometheus を公開する設定：

      {{< text bash >}}
      $ cat <<EOF | kubectl apply -f -
      apiVersion: networking.istio.io/v1
      kind: Gateway
      metadata:
      name: prometheus-gateway
      namespace: istio-system
      spec:
      selector:
      istio: ingressgateway
      servers:

      - port:
        number: 443
        name: https-prom
        protocol: HTTPS
        tls:
        mode: SIMPLE
        credentialName: telemetry-gw-cert
        hosts:
        - "prometheus.${INGRESS_DOMAIN}"

      ***

      apiVersion: networking.istio.io/v1
      kind: VirtualService
      metadata:
      name: prometheus-vs
      namespace: istio-system
      spec:
      hosts:

      - "prometheus.${INGRESS_DOMAIN}"
        gateways:
      - prometheus-gateway
        http:
      - route:
        - destination:
          host: prometheus
          port:
          number: 9090

      ***

      apiVersion: networking.istio.io/v1
      kind: DestinationRule
      metadata:
      name: prometheus
      namespace: istio-system
      spec:
      host: prometheus
      trafficPolicy:
      tls:
      mode: DISABLE

      ***

      EOF
      gateway.networking.istio.io/prometheus-gateway created
      virtualservice.networking.istio.io/prometheus-vs created
      destinationrule.networking.istio.io/prometheus created
      {{< /text >}}

   1. トレースサービスを公開する設定：

      {{< text bash >}}
      $ cat <<EOF | kubectl apply -f -
      apiVersion: networking.istio.io/v1
      kind: Gateway
      metadata:
      name: tracing-gateway
      namespace: istio-system
      spec:
      selector:
      istio: ingressgateway
      servers:

      - port:
        number: 443
        name: https-tracing
        protocol: HTTPS
        tls:
        mode: SIMPLE
        credentialName: telemetry-gw-cert
        hosts:
        - "tracing.${INGRESS_DOMAIN}"

      ***

      apiVersion: networking.istio.io/v1
      kind: VirtualService
      metadata:
      name: tracing-vs
      namespace: istio-system
      spec:
      hosts:

      - "tracing.${INGRESS_DOMAIN}"
        gateways:
      - tracing-gateway
        http:
      - route:
        - destination:
          host: tracing
          port:
          number: 80

      ***

      apiVersion: networking.istio.io/v1
      kind: DestinationRule
      metadata:
      name: tracing
      namespace: istio-system
      spec:
      host: tracing
      trafficPolicy:
      tls:
      mode: DISABLE

      ***

      EOF
      gateway.networking.istio.io/tracing-gateway created
      virtualservice.networking.istio.io/tracing-vs created
      destinationrule.networking.istio.io/tracing created
      {{< /text >}}

1. ブラウザからこれらのテレメトリアドオンにアクセスします。

   {{< warning >}}
   自己署名証明書を使用した場合、ブラウザはそれを安全でないとマークする可能性があります。
   {{< /warning >}}

   - Kiali：`https://kiali.${INGRESS_DOMAIN}`
   - Prometheus：`https://prometheus.${INGRESS_DOMAIN}`
   - Grafana：`https://grafana.${INGRESS_DOMAIN}`
   - Tracing：`https://tracing.${INGRESS_DOMAIN}`

### 方法 2：非セキュアアクセス（HTTP）{#option-two-insecure-access-HTTP}

1. テレメトリアドオンのネットワーク設定を適用します。

   1. Grafana を公開する設定：

      {{< text bash >}}
      $ cat <<EOF | kubectl apply -f -
      apiVersion: networking.istio.io/v1
      kind: Gateway
      metadata:
      name: grafana-gateway
      namespace: istio-system
      spec:
      selector:
      istio: ingressgateway
      servers:

      - port:
        number: 80
        name: http-grafana
        protocol: HTTP
        hosts:
        - "grafana.${INGRESS_DOMAIN}"

      ***

      apiVersion: networking.istio.io/v1
      kind: VirtualService
      metadata:
      name: grafana-vs
      namespace: istio-system
      spec:
      hosts:

      - "grafana.${INGRESS_DOMAIN}"
        gateways:
      - grafana-gateway
        http:
      - route:
        - destination:
          host: grafana
          port:
          number: 3000

      ***

      apiVersion: networking.istio.io/v1
      kind: DestinationRule
      metadata:
      name: grafana
      namespace: istio-system
      spec:
      host: grafana
      trafficPolicy:
      tls:
      mode: DISABLE

      ***

      EOF
      gateway.networking.istio.io/grafana-gateway created
      virtualservice.networking.istio.io/grafana-vs created
      destinationrule.networking.istio.io/grafana created
      {{< /text >}}

   1. アプリケーション Kiali を公開する設定：

      {{< text bash >}}
      $ cat <<EOF | kubectl apply -f -
      apiVersion: networking.istio.io/v1
      kind: Gateway
      metadata:
      name: kiali-gateway
      namespace: istio-system
      spec:
      selector:
      istio: ingressgateway
      servers:

      - port:
        number: 80
        name: http-kiali
        protocol: HTTP
        hosts:
        - "kiali.${INGRESS_DOMAIN}"

      ***

      apiVersion: networking.istio.io/v1
      kind: VirtualService
      metadata:
      name: kiali-vs
      namespace: istio-system
      spec:
      hosts:

      - "kiali.${INGRESS_DOMAIN}"
        gateways:
      - kiali-gateway
        http:
      - route:
        - destination:
          host: kiali
          port:
          number: 20001

      ***

      apiVersion: networking.istio.io/v1
      kind: DestinationRule
      metadata:
      name: kiali
      namespace: istio-system
      spec:
      host: kiali
      trafficPolicy:
      tls:
      mode: DISABLE

      ***

      EOF
      gateway.networking.istio.io/kiali-gateway created
      virtualservice.networking.istio.io/kiali-vs created
      destinationrule.networking.istio.io/kiali created
      {{< /text >}}

   1. Prometheus を公開する設定：

      {{< text bash >}}
      $ cat <<EOF | kubectl apply -f -
      apiVersion: networking.istio.io/v1
      kind: Gateway
      metadata:
      name: prometheus-gateway
      namespace: istio-system
      spec:
      selector:
      istio: ingressgateway
      servers:

      - port:
        number: 80
        name: http-prom
        protocol: HTTP
        hosts:
        - "prometheus.${INGRESS_DOMAIN}"

      ***

      apiVersion: networking.istio.io/v1
      kind: VirtualService
      metadata:
      name: prometheus-vs
      namespace: istio-system
      spec:
      hosts:

      - "prometheus.${INGRESS_DOMAIN}"
        gateways:
      - prometheus-gateway
        http:
      - route:
        - destination:
          host: prometheus
          port:
          number: 9090

      ***

      apiVersion: networking.istio.io/v1
      kind: DestinationRule
      metadata:
      name: prometheus
      namespace: istio-system
      spec:
      host: prometheus
      trafficPolicy:
      tls:
      mode: DISABLE

      ***

      EOF
      gateway.networking.istio.io/prometheus-gateway created
      virtualservice.networking.istio.io/prometheus-vs created
      destinationrule.networking.istio.io/prometheus created
      {{< /text >}}

   1. トレースサービスを公開する設定：

      {{< text bash >}}
      $ cat <<EOF | kubectl apply -f -
      apiVersion: networking.istio.io/v1
      kind: Gateway
      metadata:
      name: tracing-gateway
      namespace: istio-system
      spec:
      selector:
      istio: ingressgateway
      servers:

      - port:
        number: 80
        name: http-tracing
        protocol: HTTP
        hosts:
        - "tracing.${INGRESS_DOMAIN}"

      ***

      apiVersion: networking.istio.io/v1
      kind: VirtualService
      metadata:
      name: tracing-vs
      namespace: istio-system
      spec:
      hosts:

      - "tracing.${INGRESS_DOMAIN}"
        gateways:
      - tracing-gateway
        http:
      - route:
        - destination:
          host: tracing
          port:
          number: 80

      ***

      apiVersion: networking.istio.io/v1
      kind: DestinationRule
      metadata:
      name: tracing
      namespace: istio-system
      spec:
      host: tracing
      trafficPolicy:
      tls:
      mode: DISABLE

      ***

      EOF
      gateway.networking.istio.io/tracing-gateway created
      virtualservice.networking.istio.io/tracing-vs created
      destinationrule.networking.istio.io/tracing created
      {{< /text >}}

1. ブラウザからこれらのテレメトリアドオンにアクセスします。

   - Kiali：`http://kiali.${INGRESS_DOMAIN}`
   - Prometheus：`http://prometheus.${INGRESS_DOMAIN}`
   - Grafana：`http://grafana.${INGRESS_DOMAIN}`
   - Tracing：`http://tracing.${INGRESS_DOMAIN}`

## クリーンアップ {#cleanup}

- すべての関連する Gateway を削除します：

  {{< text bash >}}
  $ kubectl -n istio-system delete gateway grafana-gateway kiali-gateway prometheus-gateway tracing-gateway
  gateway.networking.istio.io "grafana-gateway" deleted
  gateway.networking.istio.io "kiali-gateway" deleted
  gateway.networking.istio.io "prometheus-gateway" deleted
  gateway.networking.istio.io "tracing-gateway" deleted
  {{< /text >}}

- すべての関連する Virtual Service を削除します：

  {{< text bash >}}
  $ kubectl -n istio-system delete virtualservice grafana-vs kiali-vs prometheus-vs tracing-vs
  virtualservice.networking.istio.io "grafana-vs" deleted
  virtualservice.networking.istio.io "kiali-vs" deleted
  virtualservice.networking.istio.io "prometheus-vs" deleted
  virtualservice.networking.istio.io "tracing-vs" deleted
  {{< /text >}}

- すべての関連する Destination Rule を削除します：

  {{< text bash >}}
  $ kubectl -n istio-system delete destinationrule grafana kiali prometheus tracing
  destinationrule.networking.istio.io "grafana" deleted
  destinationrule.networking.istio.io "kiali" deleted
  destinationrule.networking.istio.io "prometheus" deleted
  destinationrule.networking.istio.io "tracing" deleted
  {{< /text >}}
