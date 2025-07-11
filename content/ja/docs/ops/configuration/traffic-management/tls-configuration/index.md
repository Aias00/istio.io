---
title: TLS 設定の理解
linktitle: TLS 設定
description: TLS 設定を使って安全なネットワークトラフィックを構成する方法。
weight: 30
keywords: [traffic-management, proxy]
owner: istio/wg-networking-maintainers
test: n/a
---

Istio の重要な機能の 1 つは、メッシュ内のトラフィックをロックダウンし保護できることです。しかし、TLS 設定の構成は混乱しやすく、設定ミスの一般的な原因となります。
この記事では、Istio 内でリクエストを送信する際に関係するさまざまな接続と、それらの TLS 設定方法について説明します。
[TLS 設定ミス](/ja/docs/ops/common-problems/network-issues/#tls-configuration-mistakes)も参照してください。
この記事では TLS 設定に関するよくある問題もまとめています。

## Sidecar

Sidecar のトラフィックにはさまざまな接続が関係します。1 つずつ分解してみましょう。

{{< image width="100%"
    link="sidecar-connections.svg"
    alt="Sidecar プロキシネットワーク接続"
    title="Sidecar 接続"
    caption="Sidecar プロキシネットワーク接続"
    >}}

1. **外部インバウンドトラフィック**
   これは Sidecar がキャプチャする外部クライアントからのトラフィックです。
   クライアントがメッシュ外の場合、このトラフィックは Istio によって双方向 TLS で暗号化されることがあります。
   Sidecar のデフォルト設定は `PERMISSIVE`（許容）モードで、mTLS と非 mTLS の両方のトラフィックを受け入れます。
   このモードは `STRICT`（厳格）モード（mTLS のみ許可）や `DISABLE`（無効）モード（平文のみ許可）に変更できます。
   mTLS モードは [`PeerAuthentication` リソース](/ja/docs/reference/config/security/peer_authentication/)で設定します。

1. **内部インバウンドトラフィック**
   これは Sidecar からアプリケーションサービスに流れるトラフィックで、トラフィックはそのまま転送されます。
   これは常に平文という意味ではなく、Sidecar も TLS 接続を使う場合がありますが、Sidecar 内で新たな TLS 接続は発生しません。

1. **内部アウトバウンドトラフィック**
   これはアプリケーションサービスから Sidecar に送られるトラフィックで、Sidecar がインターセプトします。
   アプリケーションは平文または TLS でトラフィックを送信できます。
   [自動プロトコル選択](/ja/docs/ops/configuration/traffic-management/protocol-selection/#automatic-protocol-selection)が有効な場合、Istio はプロトコルを自動選択します。
   そうでない場合は、宛先サービスのポート名で[手動プロトコル指定](/ja/docs/ops/configuration/traffic-management/protocol-selection/#explicit-protocol-selection)が必要です。

1. **外部アウトバウンドトラフィック**
   これは Sidecar から外部宛先に出ていくトラフィックです。トラフィックはそのまま転送されるか、TLS 接続（mTLS または標準 TLS）を開始できます。
   これは [`DestinationRule` リソース](/ja/docs/reference/config/networking/destination-rule/)の
   `trafficPolicy` で TLS モードを制御します。`DISABLE` で平文、
   `SIMPLE`、`MUTUAL`、`ISTIO_MUTUAL` で TLS 接続を開始します。

ポイントは：

- `PeerAuthentication` は Sidecar が受信する mTLS トラフィックのタイプを設定します。
- `DestinationRule` は Sidecar が送信する TLS トラフィックのタイプを設定します。
- ポート名または自動プロトコル選択が、Sidecar がトラフィックのプロトコルを解釈する方法を決定します。

## 自動 mTLS {#auto-mTLS}

上記の通り、`DestinationRule` は送信トラフィックで mTLS を使うかどうかを制御します。
しかし、すべてのワークロードでこれを設定するのは面倒です。多くの場合、Istio には常に mTLS を使ってほしいでしょう。
可能な限り、メッシュ外（Sidecar を持たないワークロード）には平文のみを送信したいはずです。

Istio には「自動 mTLS」という機能があり、設定を簡単にします。自動 mTLS の仕組みは次の通りです：
`DestinationRule` で明示的に TLS 設定がされていない場合、Sidecar は自動的に
[Istio 双方向 TLS](/ja/about/faq/#difference-between-mutual-and-istio-mutual) を使うかどうかを選択します。
つまり、何も設定しなくても、メッシュ内のすべてのトラフィックは mTLS で暗号化されます。

## ゲートウェイ {#gateways}

ゲートウェイ経由のリクエストは常に 2 つの接続を持ちます：

{{< image width="100%"
    link="gateway-connections.svg"
    alt="ゲートウェイネットワーク接続"
    title="ゲートウェイ接続"
    caption="ネットワーク接続"
    >}}

1. インバウンドリクエストはクライアント（curl や Web ブラウザなど）から発信されます。これは「ダウンストリーム」接続と呼ばれます。

1. アウトバウンドリクエストはゲートウェイからバックエンドに発信されます。これは「アップストリーム」接続と呼ばれます。

この 2 つの接続はそれぞれ独立した TLS 設定を持ちます。

イングレス・エグレスゲートウェイの設定は同じです。
`istio-ingress-gateway` と `istio-egress-gateway` はカスタマイズされたゲートウェイデプロイメントです。
違いはイングレスゲートウェイのクライアントはメッシュ外、エグレスゲートウェイの宛先はメッシュ外という点です。

### インバウンド {#inbound}

インバウンドリクエストでは、ゲートウェイはトラフィックをデコードしてルーティングルールを適用する必要があります。
ゲートウェイは [`Gateway` リソース](/ja/docs/reference/config/networking/gateway/)のサービス設定に従ってデコードします。
たとえば、インバウンド接続がプレーンな HTTP の場合、ポートプロトコルは `HTTP` に設定します：

{{< text yaml >}}
apiVersion: networking.istio.io/v1
kind: Gateway
...
servers:

- port:
  number: 80
  name: http
  protocol: HTTP
  {{< /text >}}

同様に、純粋な TCP トラフィックの場合はプロトコルを `TCP` に設定します。

TLS 接続の場合はさらにオプションがあります：

1. どのプロトコルがカプセル化されていますか？
   接続が HTTPS の場合、サービスプロトコルは `HTTPS` に設定します。
   逆に、TLS でカプセル化された純粋な TCP 接続の場合はプロトコルを `TLS` にします。

1. TLS 接続は終端されますか、それともパススルーですか？
   パススルーの場合、TLS モードフィールドを `PASSTHROUGH` に設定します：

   {{< text yaml >}}
   apiVersion: networking.istio.io/v1
   kind: Gateway
   ...
   servers:

   - port:
     number: 443
     name: https
     protocol: HTTPS
     tls:
     mode: PASSTHROUGH
     {{< /text >}}

   このモードでは、Istio は SNI 情報に基づいてルーティングし、リクエストをそのまま宛先に転送します。

1. 双方向 TLS を使いますか？
   双方向 TLS は TLS モード `MUTUAL` で設定できます。設定すると、クライアント証明書は設定した
   `caCertificates` または `credentialName` でリクエスト・検証されます：

   {{< text yaml >}}
   apiVersion: networking.istio.io/v1
   kind: Gateway
   ...
   servers:

   - port:
     number: 443
     name: https
     protocol: HTTPS
     tls:
     mode: MUTUAL
     caCertificates: ...
     {{< /text >}}

### アウトバウンド {#outbound}

アウトバウンド設定は、インバウンド設定で期待されるトラフィックタイプやその処理方法に基づいて、ゲートウェイがどのタイプのトラフィックを送信するかを制御します。
TLS 設定は `DestinationRule` で行い、[Sidecar](#sidecars) の外部アウトバウンドトラフィックや、デフォルトの[自動 mTLS](#auto-mTLS)と同様です。

唯一の違いは、設定時に `Gateway` の設定を慎重に考慮する必要がある点です。たとえば、`Gateway` で TLS `PASSTHROUGH` を設定し、`DestinationRule` で TLS ソースを設定すると、
最終的に[二重暗号化](/ja/docs/ops/common-problems/network-issues/#double-tls)となります。
これは有効な設定ですが、通常の構成ではありません。

ゲートウェイにバインドされた `VirtualService` も `Gateway` の定義と[整合性を保つ](/ja/docs/ops/common-problems/network-issues/#gateway-mismatch)必要があります。
