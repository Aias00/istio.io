---
title: サードパーティ製ロードバランサー
description: Istio がサードパーティ製ロードバランサーとどのように統合できるか。
weight: 90
keywords: [traffic-management, ingress]
owner: istio/wg-networking-maintainers
test: n/a
---

Istio は Ingress とサービスメッシュの両方の実装を提供しており、これらは一緒に、または個別に使用できます。設計上はシームレスに連携しますが、サードパーティ製 Ingress と統合が必要な場合もあります。これは移行、機能要件、または好みによるものです。

## 統合モード {#integration-modes}

「スタンドアロン」モードでは、サードパーティ製 Ingress が直接バックエンドにトラフィックを送信します。この場合、バックエンドには Istio Sidecar が注入されていることがあります。

{{< mermaid >}}
graph LR
cc((クライアント))
tpi(サードパーティ Ingress)
a(バックエンド)
cc-->tpi-->a
{{< /mermaid >}}

このモードでは、ほとんどの動作は通常通りです。サービスメッシュ内のクライアントは、接続先のバックエンドに Sidecar があるかどうかを意識する必要はありません。ただし、Ingress では mTLS が使用されないため、予期しない動作が発生する可能性があります。そのため、この構成の多くは mTLS の有効化に関するものです。

「チェイン」モードでは、サードパーティ Ingress と Istio 独自の Gateway を順番に使用します。これは 2 層の機能が必要な場合に便利です。特に、クラウドマネージドロードバランサーでは、グローバルアドレスやマネージド証明書などの機能があるため有用です。

{{< mermaid >}}
graph LR
cc((クライアント))
tpi(サードパーティ Ingress)
ii(Istio Gateway)
a(バックエンド)
cc-->tpi
tpi-->ii
ii-->a
{{< /mermaid >}}

## クラウドロードバランサー {#cloud-load-balancers}

通常、クラウドロードバランサーはスタンドアロンモードで mTLS なしで動作します。チェインモードや mTLS 有効のスタンドアロンモードをサポートするには、ベンダー固有の設定が必要です。

### Google HTTP/HTTPS ロードバランサー {#google-https-load-balancer}

Google HTTP/HTTPS ロードバランサーの統合はスタンドアロンモードのみサポートされ、mTLS が不要な場合に直接利用できます（mTLS はサポートされていません）。

チェインモードも可能です。設定方法は[Google ドキュメント](https://cloud.google.com/architecture/exposing-service-mesh-apps-through-gke-ingress)を参照してください。

## クラスタ内ロードバランサー {#in-cluster-load-balancers}

通常、クラスタ内ロードバランサーはスタンドアロンモードで mTLS なしで動作します。

mTLS 有効のスタンドアロンモードを実現するには、クラスタ内ロードバランサーの Pod に Sidecar を注入します。これは標準の Sidecar 注入に加えて 2 つの手順が必要です：

1. インバウンドトラフィックのリダイレクトを無効化します。
   必須ではありませんが、通常は Sidecar で**アウトバウンド**トラフィックのみを処理したい場合に推奨されます。クライアントからのインバウンド接続はロードバランサー自体が処理します。これにより、元のクライアント IP アドレスも保持されます（Sidecar で失われることがあります）。このモードは、ロードバランサー `Pod` に `traffic.sidecar.istio.io/includeInboundPorts: ""` アノテーションを追加することで有効化できます。
1. サービスルーティングを有効化します。
   サービス宛てのリクエストでなければ、Istio Sidecar は正しく動作しません。多くのロードバランサーはデフォルトで特定の Pod IP に送信しますが、これでは mTLS が機能しません。ベンダーごとの手順は異なりますが、以下にいくつか例を示します。詳細は各ベンダーのドキュメントを参照してください。

   また、`Host` ヘッダーをサービス名に設定することも有効な場合がありますが、予期しない動作を招くことがあります。ロードバランサーは特定の Pod を選択しますが、Istio はそれを無視します。詳細は[こちら](/ja/docs/ops/configuration/traffic-management/traffic-routing/#http)を参照してください。

### ingress-nginx

`ingress-nginx` でサービスルーティングを有効にするには、`Ingress` リソースにアノテーションを追加します：

{{< text yaml >}}
nginx.ingress.kubernetes.io/service-upstream: "true"
{{< /text >}}

### Emissary-Ingress

Emissary-ingress はデフォルトでサービスルーティングを使用するため、追加の手順は不要です。
