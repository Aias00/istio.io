---
title: MUTUAL と ISTIO_MUTUAL TLS モードの違いは何ですか？
weight: 30
---

2 つの `DestinationRule` 設定は、双方向 TLS トラフィックを送信します。
`ISTIO_MUTUAL` を使用する場合、Istio 証明書が自動的に使用されます。
`MUTUAL` を使用する場合、キー、証明書、および信頼できる CA を設定する必要があります。
非 Istio アプリケーションとの双方向 TLS を許可します。
