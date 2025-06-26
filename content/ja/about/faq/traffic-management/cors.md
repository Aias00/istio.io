---
title: なぜ CORS（クロスオリジンリソース共有）設定が機能しないのですか？
weight: 40
---

[CORS（クロスオリジンリソース共有）設定](/ja/docs/reference/config/networking/virtual-service/#CorsPolicy)を適用した後、
何も起こっていないように見えることがあり、どこが問題なのかわからないかもしれません。
CORS は、設定時によく誤解される HTTP の概念です。

この問題を理解するには、後退して、
[CORS とは何か](https://developer.mozilla.org/ja/docs/Web/HTTP/CORS)、
そしていつ使用すべきかを見てみる必要があります。デフォルトでは、ブラウザはスクリプトからの "cross origin" リクエストに制限を課します。
例えば、これは、ユーザーの機密情報を盗み取るために、ウェブサイト `attack.example.com` が `bank.example.com` に JavaScript リクエストを送信することを防ぎます。

このリクエストを許可するには、`bank.example.com` が `attack.example.com` がクロスオリジンリクエストを実行することを許可する必要があります。
これが CORS の役割です。もし、Istio が有効なクラスター内で `bank.example.com` サービスを提供したい場合、
`corsPolicy` を設定することで、これを許可することができます：

{{< text yaml >}}
apiVersion: networking.istio.io/v1
kind: VirtualService
metadata:
  name: bank
spec:
  hosts:
  - bank.example.com
  http:
  - corsPolicy:
      allowOrigins:
      - exact: https://attack.example.com
...
{{< /text >}}

この場合、単一の起源を明示的に許可します。通配符は、不敏感なページに通常使用されます。

これを行った後、一般的なエラーは、
`curl bank.example.com -H "Origin: https://attack.example.com"` のようなリクエストを送信し、
このリクエストが拒否されることを期待することです。
しかし、curl や他の多くのクライアントは、CORS がブラウザの制約であるため、拒否されたリクエストを見ることはありません。
CORS 設定は、単に `Access-Control-*` ヘッダーを追加するだけです。
もし、レスポンスが不満足であれば、クライアント（ブラウザ）がリクエストを拒否します。
ブラウザでは、これは[事前リクエスト](https://developer.mozilla.org/ja/docs/Web/HTTP/CORS#preflighted_requests)によって行われます。
これは[事前リクエスト](https://developer.mozilla.org/ja/docs/Web/HTTP/CORS#preflighted_requests)によって行われます。
