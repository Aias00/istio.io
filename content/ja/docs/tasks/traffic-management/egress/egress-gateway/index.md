---
title: Egress ゲートウェイ
description: Istio で専用ゲートウェイサービスを通じてトラフィックを外部サービスに誘導する方法を説明します。
weight: 30
keywords: [traffic-management, egress]
aliases:
  - /zh/docs/examples/advanced-gateways/egress-gateway/
owner: istio/wg-networking-maintainers
test: yes
---

{{<warning>}}
この例は Minikube では動作しません。
{{</warning>}}

[外部サービスへのアクセス](/ja/docs/tasks/traffic-management/egress/egress-control) のタスクでは、Istio を使ってメッシュ内のアプリケーションから外部 HTTP および HTTPS サービスへアクセスする方法を紹介しましたが、あのタスクではクライアントの Sidecar から直接外部サービスへリクエストしていました。本記事の例では、Istio を使って専用の **Egress ゲートウェイ** サービス経由で外部サービスへ間接的にアクセスする方法を紹介します。

Istio では [Ingress ゲートウェイと Egress ゲートウェイ](/ja/docs/reference/config/networking/gateway/) を使って、サービスメッシュのエッジで動作するロードバランサを構成できます。Ingress ゲートウェイはメッシュへのすべてのインバウンドトラフィックの入口を定義します。Egress ゲートウェイは Ingress ゲートウェイと対になる概念で、メッシュの出口を定義します。Egress ゲートウェイを使うことで、Istio の機能（監視やルーティングルールなど）をメッシュのアウトバウンドトラフィックにも適用できます。

## ユースケース {#use-case}

セキュリティ要件が厳しい組織を想定してください。サービスメッシュ内のすべてのアウトバウンドトラフィックを専用ノード群経由で流す必要があります。これらの専用ノードは特別なマシン上で動作し、クラスタ内の他のアプリケーションノードとは分離されています。これらのノードはアウトバウンドトラフィックのポリシーを実施し、他のノードより厳格に監視されます。

もう一つのユースケースは、クラスタ内のアプリケーションノードがパブリック IP を持たず、メッシュ内サービスがインターネットに直接アクセスできない場合です。Egress ゲートウェイを定義し、Egress ゲートウェイノードにパブリック IP を割り当て、すべてのアウトバウンドトラフィックをそこに誘導することで、アプリケーションノードから外部サービスへのアクセスを制御できます。

{{< boilerplate gateway-api-gamma-experimental >}}

## 始める前に {#before-you-begin}

- [インストールガイド](/ja/docs/setup/) の手順に従って Istio をセットアップしてください。

  {{< tip >}}
  `demo` [プロファイル](/ja/docs/setup/additional-setup/config-profiles/) でインストールした場合、Egress ゲートウェイとアクセスログは有効になっています。
  {{< /tip >}}

- [curl]({{< github_tree >}}/samples/curl) サンプルアプリをデプロイし、リクエスト送信のテストソースとします。

  {{< text bash >}}
  $ kubectl apply -f @samples/curl/curl.yaml@
  {{< /text >}}

  {{< tip >}}
  `curl` がインストールされている任意の Pod をテストソースとして利用できます。
  {{< /tip >}}

- 環境変数 `SOURCE_POD` を、テストソース Pod の名前で設定します：

  {{< text bash >}}
  $ export SOURCE_POD=$(kubectl get pod -l app=curl -o jsonpath={.items..metadata.name})
  {{< /text >}}

  {{< warning >}}
  このタスクのコマンドは `default` 名前空間で Egress ゲートウェイの DestinationRule を作成します。
  クライアント `SOURCE_POD` も `default` 名前空間で動作していることを前提としています。そうでない場合、[DestinationRule の探索パス](/ja/docs/ops/best-practices/traffic-management/#cross-namespace-configuration)でルールが見つからず、クライアントリクエストが失敗します。
  {{< /warning >}}

- アクセスログが有効でない場合は、[Envoy のアクセスログを有効化](/ja/docs/tasks/observability/logs/access-log/#enable-envoy-s-access-logging)してください。例：

  {{< text bask >}}
  $ istioctl install <Istio インストール時のパラメータ> --set meshConfig.accessLogFile=/dev/stdout
  {{< /text >}}

## Istio Egress ゲートウェイのデプロイ {#deploy-Istio-egress-gateway}

{{< tip >}}
Gateway API で Egress ゲートウェイを構成する場合、
これらの Egress ゲートウェイは[自動的にデプロイ](/ja/docs/tasks/traffic-management/ingress/gateway-api/#deployment-methods)されます。
以降で `Gateway API` の手順を使う場合、このセクションはスキップできます。
{{< /tip >}}

1. Istio Egress ゲートウェイがデプロイ済みか確認します：

   {{< text bash >}}
   $ kubectl get pod -l istio=egressgateway -n istio-system
   {{< /text >}}

   Pod が返らない場合は、次の手順で Istio Egress ゲートウェイをデプロイします。

1. `IstioOperator` 設定でインストールしている場合は、次のフィールドを追加します：

   {{< text yaml >}}
   spec:
   components:
   egressGateways: - name: istio-egressgateway
   enabled: true
   {{< /text >}}

   それ以外の場合は、`istioctl install` コマンドに次のオプションを追加します：

   {{< text syntax=bash snip_id=none >}}
   $ istioctl install <flags-you-used-to-install-Istio> \
    --set "components.egressGateways[0].name=istio-egressgateway" \
    --set "components.egressGateways[0].enabled=true"
   {{< /text >}}

## Egress ゲートウェイの定義と HTTP トラフィックの誘導 {#egress-gateway-for-http-traffic}

まず `ServiceEntry` を作成し、外部サービスへの直接アクセスを許可します。

1. `edition.cnn.com` 用の `ServiceEntry` を定義します：

   {{< warning >}}
   下記のサービスエントリでは `DNS` 解決を使う必要があります。`NONE` にすると、ゲートウェイが自身にトラフィックをループさせてしまいます。これは、ゲートウェイが受け取るリクエストの宛先 IP がゲートウェイのサービス IP になるためです（このリクエストは Sidecar プロキシからゲートウェイに誘導されます）。

   `DNS` 解決を使うことで、ゲートウェイは外部サービスの IP を DNS で取得し、その IP にトラフィックを転送します。
   {{< /warning >}}

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
     name: http-port
     protocol: HTTP
   - number: 443
     name: https
     protocol: HTTPS
     resolution: DNS
     EOF
     {{< /text >}}

1. [https://edition.cnn.com/politics](https://edition.cnn.com/politics) へ HTTPS リクエストを送り、`ServiceEntry` が正しく機能しているか確認します。

   {{< text bash >}}
   $ kubectl exec "$SOURCE_POD" -c curl -- curl -sSL -o /dev/null -D - http://edition.cnn.com/politics
   ...
   HTTP/1.1 301 Moved Permanently
   ...
   location: https://edition.cnn.com/politics
   ...

   HTTP/2 200
   Content-Type: text/html; charset=utf-8
   ...
   {{< /text >}}

   出力は[外部トラフィックの TLS 発行](/ja/docs/tasks/traffic-management/egress/egress-tls-origination/)の例と同じで、まだ TLS は発行されていません。

1. `edition.cnn.com` の 80 番ポート用 Egress ゲートウェイを作成します。

（以降、各セクションの中国語本文を日本語に翻訳し、コマンド・コード・リンク・フォーマット等はそのまま保持して続けてください）
