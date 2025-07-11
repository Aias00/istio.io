---
title: HTTP ヘッダーへの JWT クレームのコピー
description: ユーザーが JWT クレームを HTTP ヘッダーにコピーする方法を示します。
weight: 30
keywords: [security, authentication, JWT, claim]
aliases:
  - /zh/docs/tasks/security/istio-auth.html
  - /zh/docs/tasks/security/authn-policy/
owner: istio/wg-security-maintainers
test: yes
status: Experimental
---

{{< boilerplate experimental >}}

このタスクでは、Istio のリクエスト認証ポリシーによって JWT 認証が正常に完了した後、
JWT クレームを HTTP ヘッダーにコピーする方法を紹介します。

{{< warning >}}
サポートされているのは string、boolean、integer 型のクレームのみです。現時点では array 型のクレームはサポートされていません。
{{< /warning >}}

## 始める前に {#before-you-begin}

このタスクを始める前に、以下の準備をしてください：

- [Istio エンドユーザー認証](/ja/docs/tasks/security/authentication/authn-policy/#end-user-authentication)のサポートに慣れていること。

- [Istio インストールガイド](/ja/docs/setup/install/istioctl/)を使用して Istio をインストールしていること。

- Sidecar インジェクションが有効な名前空間 `foo` に `httpbin` と `curl` のワークロードをデプロイします。以下のコマンドで名前空間とワークロードのサンプルをデプロイします：

  {{< text bash >}}
  $ kubectl create ns foo
  $ kubectl label namespace foo istio-injection=enabled
  $ kubectl apply -f @samples/httpbin/httpbin.yaml@ -n foo
  $ kubectl apply -f @samples/curl/curl.yaml@ -n foo
  {{< /text >}}

- 以下のコマンドで `curl` が `httpbin` と正常に通信できることを確認します：

  {{< text bash >}}
  $ kubectl exec "$(kubectl get pod -l app=curl -n foo -o jsonpath={.items..metadata.name})" -c curl -n foo -- curl http://httpbin.foo:8000/ip -sS -o /dev/null -w "%{http_code}\n"
  200
  {{< /text >}}

  {{< warning >}}
  期待される出力が表示されない場合は、数秒後に再試行してください。バッファリングや伝播の遅延が原因の場合があります。
  {{< /warning >}}

## 有効な JWT とリスト型クレームを持つリクエストを許可する {#allow-requests-with-valid-jwt-and-list-type-claims}

1. 以下のコマンドは、`foo` 名前空間の `httpbin` ワークロードに `jwt-example` リクエスト認証ポリシーを作成します。
   このポリシーは `testing@secure.istio.io` によって発行された JWT を受け入れ、クレーム `foo` の値を HTTP ヘッダー `X-Jwt-Claim-Foo` にコピーします：

   {{< text bash >}}
   $ kubectl apply -f - <<EOF
   apiVersion: security.istio.io/v1
   kind: RequestAuthentication
   metadata:
   name: "jwt-example"
   namespace: foo
   spec:
   selector:
   matchLabels:
   app: httpbin
   jwtRules:

   - issuer: "testing@secure.istio.io"
     jwksUri: "{{< github_file >}}/security/tools/jwt/samples/jwks.json"
     outputClaimToHeaders: - header: "x-jwt-claim-foo"
     claim: "foo"
     EOF
     {{< /text >}}

1. 無効な JWT を持つリクエストが拒否されることを確認します：

   {{< text bash >}}
   $ kubectl exec "$(kubectl get pod -l app=curl -n foo -o jsonpath={.items..metadata.name})" -c curl -n foo -- curl "http://httpbin.foo:8000/headers" -sS -o /dev/null -H "Authorization: Bearer invalidToken" -w "%{http_code}\n"
   401
   {{< /text >}}

1. `testing@secure.istio.io` によって発行され、`foo` というキーのクレームを持つ JWT を取得します。

   {{< text syntax="bash" expandlinks="false" >}}
   $ TOKEN=$(curl {{< github_file >}}/security/tools/jwt/samples/demo.jwt -s) && echo "$TOKEN" | cut -d '.' -f2 - | base64 --decode -
   {"exp":4685989700,"foo":"bar","iat":1532389700,"iss":"testing@secure.istio.io","sub":"testing@secure.istio.io"}
   {{< /text >}}

1. 有効な JWT を持つリクエストが許可されることを確認します：

   {{< text bash >}}
   $ kubectl exec "$(kubectl get pod -l app=curl -n foo -o jsonpath={.items..metadata.name})" -c curl -n foo -- curl "http://httpbin.foo:8000/headers" -sS -o /dev/null -H "Authorization: Bearer $TOKEN" -w "%{http_code}\n"
   200
   {{< /text >}}

1. リクエストに有効な HTTP ヘッダーが含まれ、そのヘッダーに JWT クレーム値があることを確認します：

   {{< text bash >}}
   $ kubectl exec "$(kubectl get pod -l app=curl -n foo -o jsonpath={.items..metadata.name})" -c curl -n foo -- curl "http://httpbin.foo:8000/headers" -sS -H "Authorization: Bearer $TOKEN" | jq '.headers["X-Jwt-Claim-Foo"][0]'
   "bar"
   {{< /text >}}

## クリーンアップ {#clean-up}

名前空間 `foo` を削除します：

{{< text bash >}}
$ kubectl delete namespace foo
{{< /text >}}
