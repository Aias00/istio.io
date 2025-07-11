---
title: JWT トークン
description: JWT トークンのアクセス制御の設定方法を紹介します。
weight: 30
keywords: [security, authorization, jwt, claim]
aliases:
  - /zh/docs/tasks/security/rbac-groups/
  - /zh/docs/tasks/security/authorization/rbac-groups/
owner: istio/wg-security-maintainers
test: yes
---

このチュートリアルでは、Istio 認可ポリシーを設定して JSON Web Token（JWT）に基づく強制アクセス制御を実現する方法を紹介します。
Istio 認可ポリシーは、JWT クレームの文字列型とリスト型の両方をサポートしています。

## 始める前に {#before-you-begin}

このタスクを始める前に、以下を完了してください：

- [Istio エンドユーザー認証タスク](/ja/docs/tasks/security/authentication/authn-policy/#end-user-authentication)を完了してください。

- [Istio 認可の概念](/ja/docs/concepts/security/#authorization)を読んでください。

- [Istio インストールガイド](/ja/docs/setup/install/istioctl/)に従って Istio をインストールしてください。

- 2 つのワークロード（`httpbin` と `curl`）を同じ名前空間（例：`foo`）にデプロイします。
  どちらのワークロードも前段に Envoy プロキシを持ちます。以下のコマンドでデプロイできます：

  {{< text bash >}}
  $ kubectl create ns foo
  $ kubectl apply -f <(istioctl kube-inject -f @samples/httpbin/httpbin.yaml@) -n foo
  $ kubectl apply -f <(istioctl kube-inject -f @samples/curl/curl.yaml@) -n foo
  {{< /text >}}

- 次のコマンドで `curl` から `httpbin` サービスへのアクセスが正常であることを確認します：

  {{< text bash >}}
  $ kubectl exec "$(kubectl get pod -l app=curl -n foo -o jsonpath={.items..metadata.name})" -c curl -n foo -- curl http://httpbin.foo:8000/ip -sS -o /dev/null -w "%{http_code}\n"
  200
  {{< /text >}}

{{< warning >}}
期待される出力が表示されない場合は、数秒後に再試行してください。キャッシュやポリシー伝播の遅延が原因となる場合があります。
{{< /warning >}}

## 有効な JWT およびリスト型クレームを含むリクエストを許可する {#allow-requests-with-jwt-and-claims}

1. 次のコマンドで、`foo` 名前空間の `httpbin` ワークロードに `jwt-example` という認証ポリシーを作成します。
   このポリシーにより、`httpbin` ワークロードは Issuer が `testing@secure.istio.io` の JWT トークンを受け入れます：

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
     EOF
     {{< /text >}}

1. 無効な JWT を使ったリクエストが拒否されることを確認します：

   {{< text bash >}}
   $ kubectl exec "$(kubectl get pod -l app=curl -n foo -o jsonpath={.items..metadata.name})" -c curl -n foo -- curl "http://httpbin.foo:8000/headers" -sS -o /dev/null -H "Authorization: Bearer invalidToken" -w "%{http_code}\n"
   401
   {{< /text >}}

1. JWT トークンがないリクエストが許可されることを確認します（上記のポリシーには認可ポリシーが含まれていません）：

   {{< text bash >}}
   $ kubectl exec "$(kubectl get pod -l app=curl -n foo -o jsonpath={.items..metadata.name})" -c curl -n foo -- curl "http://httpbin.foo:8000/headers" -sS -o /dev/null -w "%{http_code}\n"
   200
   {{< /text >}}

1. 次のコマンドで、`foo` 名前空間の `httpbin` ワークロードに `require-jwt` という認可ポリシーを作成します。
   このポリシーは、`httpbin` サービスへのすべてのリクエストに対して、`requestPrincipal` が `testing@secure.istio.io/testing@secure.istio.io` である有効な JWT を要求します。Istio は JWT トークンの `iss` と `sub` を `/` で連結して `requestPrincipal` フィールドを生成します。

   {{< text syntax="bash" expandlinks="false" >}}
   $ kubectl apply -f - <<EOF
   apiVersion: security.istio.io/v1
   kind: AuthorizationPolicy
   metadata:
   name: require-jwt
   namespace: foo
   spec:
   selector:
   matchLabels:
   app: httpbin
   action: ALLOW
   rules:

   - from: - source:
     requestPrincipals: ["testing@secure.istio.io/testing@secure.istio.io"]
     EOF
     {{< /text >}}

1. `iss` と `sub` がともに `testing@secure.istio.io` の JWT を取得します。これにより Istio が生成する
   `requestPrincipal` の値は `testing@secure.istio.io/testing@secure.istio.io` となります：

   {{< text syntax="bash" expandlinks="false" >}}
   $ TOKEN=$(curl {{< github_file >}}/security/tools/jwt/samples/demo.jwt -s) && echo "$TOKEN" | cut -d '.' -f2 - | base64 --decode
   {"exp":4685989700,"foo":"bar","iat":1532389700,"iss":"testing@secure.istio.io","sub":"testing@secure.istio.io"}
   {{< /text >}}

1. 有効な JWT を使ったリクエストが許可されることを確認します：

   {{< text bash >}}
   $ kubectl exec "$(kubectl get pod -l app=curl -n foo -o jsonpath={.items..metadata.name})" -c curl -n foo -- curl "http://httpbin.foo:8000/headers" -sS -o /dev/null -H "Authorization: Bearer $TOKEN" -w "%{http_code}\n"
   200
   {{< /text >}}

1. JWT なしのリクエストが拒否されることを確認します：

   {{< text bash >}}
   $ kubectl exec "$(kubectl get pod -l app=curl -n foo -o jsonpath={.items..metadata.name})" -c curl -n foo -- curl "http://httpbin.foo:8000/headers" -sS -o /dev/null -w "%{http_code}\n"
   403
   {{< /text >}}

1. 次のコマンドで `require-jwt` 認可ポリシーを更新し、JWT に `groups` という名前で値が `group1` のクレームが含まれていることも要求します：

   {{< text syntax="bash" expandlinks="false" >}}
   $ kubectl apply -f - <<EOF
   apiVersion: security.istio.io/v1
   kind: AuthorizationPolicy
   metadata:
   name: require-jwt
   namespace: foo
   spec:
   selector:
   matchLabels:
   app: httpbin
   action: ALLOW
   rules:

   - from: - source:
     requestPrincipals: ["testing@secure.istio.io/testing@secure.istio.io"]
     when: - key: request.auth.claims[groups]
     values: ["group1"]
     EOF
     {{< /text >}}

   {{< warning >}}
   クレーム自体に引用符が含まれていない限り、`request.auth.claims` フィールドに引用符を含めないでください。
   {{< /warning >}}

1. `groups` クレームリストが `group1` と `group2` である JWT を取得します：

   {{< text syntax="bash" expandlinks="false" >}}
   $ TOKEN_GROUP=$(curl {{< github_file >}}/security/tools/jwt/samples/groups-scope.jwt -s) && echo "$TOKEN_GROUP" | cut -d '.' -f2 - | base64 --decode -
   {"exp":3537391104,"groups":["group1","group2"],"iat":1537391104,"iss":"testing@secure.istio.io","scope":["scope1","scope2"],"sub":"testing@secure.istio.io"}
   {{< /text >}}

1. JWT を含み、かつ JWT に `groups` クレームで値が `group1` のものが含まれるリクエストが許可されることを確認します：

   {{< text bash >}}
   $ kubectl exec "$(kubectl get pod -l app=curl -n foo -o jsonpath={.items..metadata.name})" -c curl -n foo -- curl "http://httpbin.foo:8000/headers" -sS -o /dev/null -H "Authorization: Bearer $TOKEN_GROUP" -w "%{http_code}\n"
   200
   {{< /text >}}

1. JWT を含むが `groups` クレームが含まれないリクエストが拒否されることを確認します：

   {{< text bash >}}
   $ kubectl exec "$(kubectl get pod -l app=curl -n foo -o jsonpath={.items..metadata.name})" -c curl -n foo -- curl "http://httpbin.foo:8000/headers" -sS -o /dev/null -H "Authorization: Bearer $TOKEN" -w "%{http_code}\n"
   403
   {{< /text >}}

## クリーンアップ {#cleanup}

`foo` 名前空間を削除します：

{{< text bash >}}
$ kubectl delete namespace foo
{{< /text >}}
