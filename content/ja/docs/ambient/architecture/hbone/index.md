---
title: HBONE
description: Istio の安全なトンネルプロトコルについて理解します。
weight: 2
owner: istio/wg-networking-maintainers
test: no
---

**HBONE**（HTTP Based Overlay Network Encapsulation，HTTP ベースのオーバーレイネットワークカプセル化）
は Istio コンポーネント間で使用される安全なトンネルプロトコルです。
HBONE は Istio 固有の用語です。
HBONE は、単一の mTLS 暗号化ネットワーク接続（暗号化トンネル）を使用して、
異なるアプリケーション接続に関連する TCP ストリームを透過的に多重化するメカニズムです。

Istio の現在の実装では、HBONE プロトコルには 3 つのオープンスタンダードが含まれています：

- [HTTP/2](https://httpwg.org/specs/rfc7540.html)
- [HTTP CONNECT](https://developer.mozilla.org/en-US/docs/Web/HTTP/Methods/CONNECT)
- [双向 TLS（mTLS）](https://datatracker.ietf.org/doc/html/rfc8446)

HTTP CONNECT はトンネル接続を確立するために使用され、mTLS は接続を保護および暗号化するために使用され、
HTTP/2 は単一の安全かつ暗号化されたトンネル上でアプリケーション接続ストリームを多重化し、
他のストリームレベルのメタデータを伝送するために使用されます。

## セキュリティとテナンシー {#security-and-tenancy}

mTLS 仕様の強制要件により、各下位トンネル接続は一意の送信元アイデンティティと一意の送信先アイデンティティを持ち、
それらのアイデンティティを使用して接続を暗号化する必要があります。

これは、HBONE プロトコルを使用して同じ送信先アイデンティティに対するアプリケーション接続が、
暗号化された安全な下位 HTTP/2 接続上で多重化されます - 実際には、
各一意の送信元と送信先は、自己専用の安全なトンネル接続を取得する必要があります。
その下位専用接続が複数のアプリケーションレベルの接続を処理している場合でも同様です。

## 実装の詳細 {#implementation-details}

Istio の約束により、ztunnel と HBONE プロトコルを理解する他のプロキシは TCP ポート 15008 でリスナーを公開します。

HBONE は HTTP/2、HTTP CONNECT、および mTLS の組み合わせにすぎないため、
HBONE トンネルデータパケットは以下のようになります：

{{< image width="100%"
link="hbone-packet.svg"
caption="HBONE L3 パケット形式"
>}}

HBONE トンネルの重要な特性は、アプリケーションリクエストを透過的にプロキシすることです。
これは、アプリケーショントラフィックフローを変更することなく、
接続に関するメタデータをターゲットプロキシに伝送できることを意味します。
例えば、Istio 固有のヘッダーをアプリケーショントラフィックに追加する必要はありません。

Ambient モードと標準の発展に伴い、HBONE と HTTP トンネル（例えば UDP）の他のユースケースについても将来検討されます。
