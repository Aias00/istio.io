---
title: JWT クレームに基づくルーティング
description: JWT クレームに基づいてリクエストをルーティングする Istio 認証ポリシーの使い方を示します。
weight: 10
keywords: [security, authentication, jwt, route]
owner: istio/wg-security-maintainers
test: yes
status: Alpha
---

{{< boilerplate alpha >}}

このタスクでは、Istio イングレスゲートウェイ上で JWT クレームに基づいてリクエストをルーティングする方法を、
リクエスト認証とバーチャルサービスを使って紹介します。

注意：この機能は Istio イングレスゲートウェイのみをサポートしており、JWT クレームに基づく正しい認証とルーティングにはリクエスト認証とバーチャルサービスの両方が必要です。

## 始める前に {#before-you-begin}

- Istio の[認証ポリシー](/ja/docs/concepts/security/#authentication-policies)と[バーチャルサービス](/ja/docs/concepts/traffic-management/#virtual-services)の概念を理解していること。

- [Istio インストールガイド](/ja/docs/setup/install/istioctl/)を使って Istio をインストールしていること。

- `foo` 名前空間に `httpbin` ワークロードをデプロイし、
  Istio イングレスゲートウェイ経由で以下のコマンドで公開します：

  {{< text bash >}}
  $ kubectl create ns foo
  $ kubectl apply -f <(istioctl kube-inject -f @samples/httpbin/httpbin.yaml@) -n foo
  $ kubectl apply -f @samples/httpbin/httpbin-gateway.yaml@ -n foo
  {{< /text >}}

- [イングレスの IP とポートの決定](/ja/docs/tasks/traffic-management/ingress/ingress-control/#determining-the-ingress-ip-and-ports)
  の手順に従って `INGRESS_HOST` と `INGRESS_PORT` 環境変数を定義します。

- 以下のコマンドで `httpbin` ワークロードとイングレスゲートウェイが正常に動作していることを確認します：

  {{< text bash >}}
  $ curl "$INGRESS_HOST:$INGRESS_PORT"/headers -s -o /dev/null -w "%{http_code}\n"
  200
  {{< /text >}}

{{< warning >}}
期待される出力が表示されない場合は、数秒後に再試行してください。キャッシュや転送のオーバーヘッドにより遅延が発生することがあります。
{{< /warning >}}

## JWT クレームに基づくイングレスルーティングの設定 {#configuring-ingress-routing-based-on-JWT-claims}

Istio イングレスゲートウェイは、認証済み JWT に基づくルーティングをサポートしています。
これはエンドユーザーのアイデンティティに基づくルーティングに便利で、
認証されていない HTTP 属性（パスやヘッダーなど）を使うよりも安全です。

1. JWT クレームに基づくルーティングのため、まずリクエスト認証を作成して JWT 検証を有効にします：

   {{< text bash >}}
   $ kubectl apply -f - <<EOF
   apiVersion: security.istio.io/v1
   kind: RequestAuthentication
   metadata:
   name: ingress-jwt
   namespace: istio-system
   spec:
   selector:
   matchLabels:
   istio: ingressgateway
   jwtRules:

   - issuer: "testing@secure.istio.io"
     jwksUri: "{{< github_file >}}/security/tools/jwt/samples/jwks.json"
     EOF
     {{< /text >}}

   このリクエスト認証は Istio ゲートウェイ上で JWT 検証を有効にし、
   検証済みの JWT クレームを後でバーチャルサービスのルーティングに利用できるようにします。

   このリクエスト認証はイングレスゲートウェイのみに適用されます。JWT クレームに基づくルーティングはイングレスゲートウェイでのみサポートされています。

   注意：リクエスト認証はリクエストに JWT が存在するかどうかのみをチェックします。JWT を必須にし、
   JWT がない場合にリクエストを拒否したい場合は、[タスク](/ja/docs/tasks/security/authentication/authn-policy#require-a-valid-token)で指定されている認可ポリシーを適用してください。

1. 検証済みの JWT クレームに基づいてバーチャルサービスをルーティングするように更新します：

   {{< text bash >}}
   $ kubectl apply -f - <<EOF
   apiVersion: networking.istio.io/v1
   kind: VirtualService
   metadata:
   name: httpbin
   namespace: foo
   spec:
   hosts:

   - "\*"
     gateways:
   - httpbin-gateway
     http:
   - match: - uri:
     prefix: /headers
     headers:
     "@request.auth.claims.groups":
     exact: group1
     route: - destination:
     port:
     number: 8000
     host: httpbin
     EOF
     {{< /text >}}

   バーチャルサービスは予約済みヘッダー "@request.auth.claims.groups" を使って JWT クレームの `groups` をマッチします。
   先頭の `@` は、HTTP ヘッダーではなく JWT 検証からのメタデータとマッチすることを示します。
   JWT は文字列型クレーム、文字列リスト、ネストされたクレームをサポートします。ネストされたクレーム名の区切りには `.` を使います。
   例："@request.auth.claims.name.givenName" はネストされた `name` と `givenName` クレームにマッチします。
   現在、クレーム名に `.` を含めることはサポートされていません。

## JWT クレームに基づくイングレスルーティングの検証 {#validating-ingress-routing-based-on-JWT-claims}

1. イングレスゲートウェイが JWT なしで HTTP 404 コードを返すことを確認します：

   {{< text bash >}}
   $ curl -s -I "http://$INGRESS_HOST:$INGRESS_PORT/headers"
   HTTP/1.1 404 Not Found
   ...
   {{< /text >}}

   JWT がない場合に HTTP 403 コードで明示的にリクエストを拒否したい場合は、認可ポリシーを作成できます。

1. イングレスゲートウェイが無効な JWT で HTTP 401 コードを返すことを確認します：

   {{< text bash >}}
   $ curl -s -I "http://$INGRESS_HOST:$INGRESS_PORT/headers" -H "Authorization: Bearer some.invalid.token"
   HTTP/1.1 401 Unauthorized
   ...
   {{< /text >}}

   401 はリクエスト認証によるもので、JWT クレームの検証に失敗した場合に返されます。

1. `groups: group1` クレームを含む有効な JWT トークンでイングレスゲートウェイルーティングを検証します：

   {{< text syntax="bash" expandlinks="false" >}}
   $ TOKEN_GROUP=$(curl {{< github_file >}}/security/tools/jwt/samples/groups-scope.jwt -s) && echo "$TOKEN_GROUP" | cut -d '.' -f2 - | base64 --decode
   {"exp":3537391104,"groups":["group1","group2"],"iat":1537391104,"iss":"testing@secure.istio.io","scope":["scope1","scope2"],"sub":"testing@secure.istio.io"}
   {{< /text >}}

   {{< text bash >}}
   $ curl -s -I "http://$INGRESS_HOST:$INGRESS_PORT/headers" -H "Authorization: Bearer $TOKEN_GROUP"
   HTTP/1.1 200 OK
   ...
   {{< /text >}}

1. `groups: group1` クレームを含まない有効な JWT でイングレスゲートウェイが HTTP 404 コードを返すことを確認します：

   {{< text syntax="bash" >}}
   $ TOKEN_NO_GROUP=$(curl {{< github_file >}}/security/tools/jwt/samples/demo.jwt -s) && echo "$TOKEN_NO_GROUP" | cut -d '.' -f2 - | base64 --decode
   {"exp":4685989700,"foo":"bar","iat":1532389700,"iss":"testing@secure.istio.io","sub":"testing@secure.istio.io"}
   {{< /text >}}

   {{< text bash >}}
   $ curl -s -I "http://$INGRESS_HOST:$INGRESS_PORT/headers" -H "Authorization: Bearer $TOKEN_NO_GROUP"
   HTTP/1.1 404 Not Found
   ...
   {{< /text >}}

## クリーンアップ {#cleanup}

- `foo` という名前空間を削除します：

  {{< text bash >}}
  $ kubectl delete namespace foo
  {{< /text >}}

- 認証を削除します：

  {{< text bash >}}
  $ kubectl delete requestauthentication ingress-jwt -n istio-system
  {{< /text >}}
