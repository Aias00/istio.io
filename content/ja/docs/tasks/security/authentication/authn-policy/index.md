---
title: 認証ポリシー
description: Istio の認証ポリシーを使用して双方向 TLS と基本的なエンドユーザー認証を設定する方法を示します。
weight: 10
keywords: [security,authentication]
aliases:
  - /ja/docs/tasks/security/istio-auth.html
  - /ja/docs/tasks/security/authn-policy/
owner: istio/wg-security-maintainers
test: yes
---

このタスクでは、Istio の認証ポリシーを使用して双方向 TLS と基本的なエンドユーザー認証を設定する方法を紹介します。
詳細な基本概念については、[認証の概要](/ja/docs/concepts/security/#authentication)を参照してください。

## 始める前に {#before-you-begin}

* Istio [認証ポリシー](/ja/docs/concepts/security/#authentication-policies)と[双方向 TLS 認証](/ja/docs/concepts/security/#mutual-TLS-authentication)の概念を理解していること。
* [インストール手順](/ja/docs/setup/getting-started)に従って、Kubernetes クラスターに Istio を `default` 設定テンプレートでインストールしていること。

{{< text bash >}}
$ istioctl install --set profile=default
{{< /text >}}

### 设置 {#setup}

本例中は、`foo` と `bar` の名前空間にそれぞれ Envoy プロキシ（Sidecar）付きの
`httpbin` と `curl` サービスを作成します。また、`legacy` の名前空間には Envoy プロキシ（Sidecar）なしの
`httpbin` と `curl` サービスを作成します。同じサンプルを使用してこれらのタスクを完了する場合は、
以下のコマンドを実行してください：

{{< text bash >}}
$ kubectl create ns foo
$ kubectl apply -f <(istioctl kube-inject -f @samples/httpbin/httpbin.yaml@) -n foo
$ kubectl apply -f <(istioctl kube-inject -f @samples/curl/curl.yaml@) -n foo
$ kubectl create ns bar
$ kubectl apply -f <(istioctl kube-inject -f @samples/httpbin/httpbin.yaml@) -n bar
$ kubectl apply -f <(istioctl kube-inject -f @samples/curl/curl.yaml@) -n bar
$ kubectl create ns legacy
$ kubectl apply -f @samples/httpbin/httpbin.yaml@ -n legacy
$ kubectl apply -f @samples/curl/curl.yaml@ -n legacy
{{< /text >}}

これで、`foo`、`bar` または `legacy` のいずれかの名前空間にある任意の `curl` Pod で、
`curl` を使用して `httpbin.foo`、`httpbin.bar` または `httpbin.legacy` に HTTP リクエストを送信し、
デプロイ結果を検証できます。すべてのリクエストは成功し、HTTP 200 を返す必要があります。

例えば、`curl.bar` から `httpbin.foo` への到達可能性を確認するコマンドは次のようになります：

{{< text bash >}}
$ kubectl exec "$(kubectl get pod -l app=curl -n bar -o jsonpath={.items..metadata.name})" -c curl -n bar -- curl http://httpbin.foo:8000/ip -s -o /dev/null -w "%{http_code}\n"
200
{{< /text >}}

また、以下のコマンドを使用して、すべての可能な組み合わせを確認することもできます：

{{< text bash >}}
$ for from in "foo" "bar" "legacy"; do for to in "foo" "bar" "legacy"; do kubectl exec "$(kubectl get pod -l app=curl -n ${from} -o jsonpath={.items..metadata.name})" -c curl -n ${from} -- curl -s "http://httpbin.${to}:8000/ip" -s -o /dev/null -w "curl.${from} to httpbin.${to}: %{http_code}\n"; done; done
curl.foo to httpbin.foo: 200
curl.foo to httpbin.bar: 200
curl.foo to httpbin.legacy: 200
curl.bar to httpbin.foo: 200
curl.bar to httpbin.bar: 200
curl.bar to httpbin.legacy: 200
curl.legacy to httpbin.foo: 200
curl.legacy to httpbin.bar: 200
curl.legacy to httpbin.legacy: 200
{{< /text >}}

以下のコマンドを使用して、システム内に対等認証ポリシーがないことを確認します：

{{< text bash >}}
$ kubectl get peerauthentication --all-namespaces
No resources found
{{< /text >}}

最後に、サンプルサービスがターゲットルール（destination rule）を適用していないことを確認することも重要です。
現在のターゲットルールの `host:` 値を確認し、それらが一致しないことを確認します。例えば：

{{< text bash >}}
$ kubectl get destinationrules.networking.istio.io --all-namespaces -o yaml | grep "host:"
{{< /text >}}

{{< tip >}}
ターゲットルールが表示された他のホストを設定している場合がありますが、これは Istio のバージョンによって異なります。
ただし、`foo`、`bar`、`legacy` の名前空間には、ホストに関連するターゲットルールがないはずです。
また、すべてのワイルドカード `*` に一致するように設定してはいけません。
{{< /tip >}}

## 自動双方向 TLS {#auto-mutual-TLS}

デフォルトでは、Istio は Istio プロキシ付きのサービスワークロードを追跡し、
クライアントプロキシを設定してこれらのワークロードに対して双方向 TLS トラフィックを自動的に送信し、
Sidecar なしのワークロードに対しては明文トラフィックを送信します。

したがって、プロキシ付きのワークロード間のすべてのトラフィックで双方向 TLS を有効にするには、
追加の操作は必要ありません。
例えば、リクエスト `httpbin/header` の応答を確認する必要はありません。
双方向 TLS を使用する場合、プロキシは後端の上流リクエストに `X-Forwarded-Client-Cert` ヘッダーを注入します。
このヘッダーの存在は双方向 TLS が有効であることを示しています。例えば：

{{< text bash >}}
$ kubectl exec "$(kubectl get pod -l app=curl -n foo -o jsonpath={.items..metadata.name})" -c curl -n foo -- curl -s http://httpbin.foo:8000/headers -s | jq '.headers["X-Forwarded-Client-Cert"][0]' | sed 's/Hash=[a-z0-9]*;/Hash=<redacted>;/'
  "By=spiffe://cluster.local/ns/foo/sa/httpbin;Hash=<redacted>;Subject=\"\";URI=spiffe://cluster.local/ns/foo/sa/curl"
{{< /text >}}

Sidecar なしのサーバーの場合、`X-Forwarded-Client-Cert` ヘッダーは存在しません。
これは明文リクエストを示しています。

{{< text bash >}}
$ kubectl exec "$(kubectl get pod -l app=curl -n foo -o jsonpath={.items..metadata.name})" -c curl -n foo -- curl http://httpbin.legacy:8000/headers -s | grep X-Forwarded-Client-Cert
{{< /text >}}

## グローバルに STRICT モードで Istio 双方向 TLS を有効にする {#globally-enabling-Istio-mutual-TLS-in-STRICT-mode}

Istio は自動的にプロキシ付きのワークロード間のすべてのトラフィックを双方向 TLS にアップグレードするため、
ワークロードは依然として明文トラフィックを受信できます。
グローバルな対等認証ポリシーを `STRICT` モードに設定して、
グローバルな対等認証ポリシーを `STRICT` モードに設定して、
メッシュ全体のピア認証ポリシーを `STRICT` モードに設定する必要があります。
作用域がメッシュ全体のピア認証ポリシーには `selector` を設定してはいけません。
このような認証ポリシーは**ルート名前空間**に適用する必要があります。例えば：

{{< text bash >}}
$ kubectl apply -f - <<EOF
apiVersion: security.istio.io/v1
kind: PeerAuthentication
metadata:
  name: "default"
  namespace: "istio-system"
spec:
  mtls:
    mode: STRICT
EOF
{{< /text >}}

{{< tip >}}
この例では、名前空間 `istio-system` がルート名前空間であると仮定しています。
インストール時に異なる値を使用した場合は、`istio-system` を使用した値に置き換えてください。
{{< /tip >}}

このピア認証ポリシーは、ワークロードを TLS 暗号化されたリクエストのみを受け入れるように設定します。
`selector` フィールドに値が指定されていないため、このポリシーはメッシュ内のすべてのワークロードに適用されます。

再びテストコマンドを実行します：

{{< text bash >}}
$ for from in "foo" "bar" "legacy"; do for to in "foo" "bar" "legacy"; do kubectl exec "$(kubectl get pod -l app=curl -n ${from} -o jsonpath={.items..metadata.name})" -c curl -n ${from} -- curl "http://httpbin.${to}:8000/ip" -s -o /dev/null -w "curl.${from} to httpbin.${to}: %{http_code}\n"; done; done
curl.foo to httpbin.foo: 200
curl.foo to httpbin.bar: 200
curl.foo to httpbin.legacy: 200
curl.bar to httpbin.foo: 200
curl.bar to httpbin.bar: 200
curl.bar to httpbin.legacy: 200
curl.legacy to httpbin.foo: 000
command terminated with exit code 56
curl.legacy to httpbin.bar: 000
command terminated with exit code 56
curl.legacy to httpbin.legacy: 200
{{< /text >}}

Sidecar なしのサービス（`curl.legacy`）から Sidecar 付きのサービス（`httpbin.foo` または `httpbin.bar`）へのリクエスト以外は、
他のリクエストも成功します。
これは予想通りの結果です。なぜなら、現在は双方向 TLS を厳密に要求しているため、
Sidecar なしのワークロードはこの要件を満たすことができないからです。

### 第 1 部のクリーンアップ {#cleanup-part-1}

グローバル認証ポリシーを削除します：

{{< text bash >}}
$ kubectl delete peerauthentication -n istio-system default
{{< /text >}}

## 各命名空間またはワークロードに対して双方向 TLS を有効にする {#enable-mutual-TLS-per-namespace-or-workload}

### 命名空间级别策略 {#namespace-wide-policy}

特定の名前空間内のすべてのワークロードの双方向 TLS を変更する場合は、名前空間レベルのポリシーを使用します。
このポリシーの仕様は、メッシュ全体の仕様と同じですが、`metadata` フィールドで名前空間の名前を指定できます。
例えば、以下のピア認証ポリシーは `foo` 名前空間で双方向 TLS を厳密に有効にしています：

{{< text bash >}}
$ kubectl apply -f - <<EOF
apiVersion: security.istio.io/v1
kind: PeerAuthentication
metadata:
  name: "default"
  namespace: "foo"
spec:
  mtls:
    mode: STRICT
EOF
{{< /text >}}

これらのポリシーは、名前空間 `foo` 内のサービスにのみ適用されるため、
Sidecar なしのクライアント（`curl.legacy`）から Sidecar 付きのクライアント（`httpbin.foo`）へのリクエストのみが失敗することがわかります。

{{< text bash >}}
$ for from in "foo" "bar" "legacy"; do for to in "foo" "bar" "legacy"; do kubectl exec "$(kubectl get pod -l app=curl -n ${from} -o jsonpath={.items..metadata.name})" -c curl -n ${from} -- curl "http://httpbin.${to}:8000/ip" -s -o /dev/null -w "curl.${from} to httpbin.${to}: %{http_code}\n"; done; done
curl.foo to httpbin.foo: 200
curl.foo to httpbin.bar: 200
curl.foo to httpbin.legacy: 200
curl.bar to httpbin.foo: 200
curl.bar to httpbin.bar: 200
curl.bar to httpbin.legacy: 200
curl.legacy to httpbin.foo: 000
command terminated with exit code 56
curl.legacy to httpbin.bar: 200
curl.legacy to httpbin.legacy: 200
{{< /text >}}

### 各ワークロードに対して双方向 TLS を有効にする {#enable-mutual-TLS-per-workload}

特定のワークロードに対してピア認証ポリシーを設定するには、`selector` フィールドを設定し、
必要なワークロードに一致するラベルを指定する必要があります。
例えば、以下のピア認証ポリシーとターゲットルールは `httpbin.bar` サービスを双方向 TLS を厳密に有効にします：

{{< text bash >}}
$ cat <<EOF | kubectl apply -n bar -f -
apiVersion: security.istio.io/v1
kind: PeerAuthentication
metadata:
  name: "httpbin"
  namespace: "bar"
spec:
  selector:
    matchLabels:
      app: httpbin
  mtls:
    mode: STRICT
EOF
{{< /text >}}

再びテストコマンドを実行します。予想通り、`curl.legacy` から `httpbin.bar`
へのリクエストは同じ理由で失敗します。

{{< text bash >}}
$ for from in "foo" "bar" "legacy"; do for to in "foo" "bar" "legacy"; do kubectl exec "$(kubectl get pod -l app=curl -n ${from} -o jsonpath={.items..metadata.name})" -c curl -n ${from} -- curl "http://httpbin.${to}:8000/ip" -s -o /dev/null -w "curl.${from} to httpbin.${to}: %{http_code}\n"; done; done
curl.foo to httpbin.foo: 200
curl.foo to httpbin.bar: 200
curl.foo to httpbin.legacy: 200
curl.bar to httpbin.foo: 200
curl.bar to httpbin.bar: 200
curl.bar to httpbin.legacy: 200
curl.legacy to httpbin.foo: 000
command terminated with exit code 56
curl.legacy to httpbin.bar: 000
command terminated with exit code 56
curl.legacy to httpbin.legacy: 200
{{< /text >}}

{{< text plain >}}
...
curl.legacy to httpbin.bar: 000
command terminated with exit code 56
{{< /text >}}

各ポートの双方向 TLS 設定を最適化するには、`portLevelMtls` フィールドを設定する必要があります。
例えば、以下のピア認証ポリシーは `8080` ポートを除くすべてのポートで双方向 TLS を要求します：

{{< text bash >}}
$ cat <<EOF | kubectl apply -n bar -f -
apiVersion: security.istio.io/v1
kind: PeerAuthentication
metadata:
  name: "httpbin"
  namespace: "bar"
spec:
  selector:
    matchLabels:
      app: httpbin
  mtls:
    mode: STRICT
  portLevelMtls:
    8080:
      mode: DISABLE
EOF
{{< /text >}}

1. ピア認証ポリシーのポート値はコンテナのポートです。ターゲットルールの値はサービスのポートです。
1. ポートがサービスにバインドされている場合は、`portLevelMtls` 設定のみを使用できます。その他の設定は Istio によって無視されます。

{{< text bash >}}
$ for from in "foo" "bar" "legacy"; do for to in "foo" "bar" "legacy"; do kubectl exec "$(kubectl get pod -l app=curl -n ${from} -o jsonpath={.items..metadata.name})" -c curl -n ${from} -- curl "http://httpbin.${to}:8000/ip" -s -o /dev/null -w "curl.${from} to httpbin.${to}: %{http_code}\n"; done; done
curl.foo to httpbin.foo: 200
curl.foo to httpbin.bar: 200
curl.foo to httpbin.legacy: 200
curl.bar to httpbin.foo: 200
curl.bar to httpbin.bar: 200
curl.bar to httpbin.legacy: 200
curl.legacy to httpbin.foo: 000
command terminated with exit code 56
curl.legacy to httpbin.bar: 200
curl.legacy to httpbin.legacy: 200
{{< /text >}}

### ポリシーの優先順位 {#policy-precedence}

特定のサービスポリシーが名前空間範囲のポリシーよりも優先順位が高いことを示すために、
`httpbin.foo` に双方向 TLS を無効にするポリシーを追加できます。
注意、すべての名前空間 `foo` 内のサービスに対して名前空間範囲のポリシーを作成して双方向 TLS を有効にしましたが、
`curl.legacy` から `httpbin.foo` へのリクエストは失敗します（上記参照）。

{{< text bash >}}
$ cat <<EOF | kubectl apply -n foo -f -
apiVersion: security.istio.io/v1
kind: PeerAuthentication
metadata:
  name: "overwrite-example"
  namespace: "foo"
spec:
  selector:
    matchLabels:
      app: httpbin
  mtls:
    mode: DISABLE
EOF
{{< /text >}}

`curl.legacy` からのリクエストを再実行すると、リクエストが成功し、200 コードが返されることがわかります。
これは、特定のサービスポリシーが名前空間範囲のポリシーを上書きしていることを示しています。

{{< text bash >}}
$ kubectl exec "$(kubectl get pod -l app=curl -n legacy -o jsonpath={.items..metadata.name})" -c curl -n legacy -- curl http://httpbin.foo:8000/ip -s -o /dev/null -w "%{http_code}\n"
200
{{< /text >}}

### 第 2 部のクリーンアップ {#cleanup-part-2}

前の手順で作成したポリシーを削除します：

{{< text bash >}}
$ kubectl delete peerauthentication default overwrite-example -n foo
$ kubectl delete peerauthentication httpbin -n bar
{{< /text >}}

## エンドユーザー認証 {#end-user-authentication}

この機能を体験するには、有効な JWT が必要です。
この JWT は、このチュートリアルで使用した JWKS エンドポイントに対応している必要があります。
このチュートリアルでは、Istio コードベースの
[JWT test]({{< github_file >}}/security/tools/jwt/samples/demo.jwt) 和
[JWKS endpoint]({{< github_file >}}/security/tools/jwt/samples/jwks.json)。

同様に、`httpbin.foo` を Ingress ゲートウェイで公開します
（詳細は [Ingress タスク](/ja/docs/tasks/traffic-management/ingress/)を参照してください）。

{{< boilerplate gateway-api-support >}}

{{< tabset category-name="config-api" >}}

{{< tab name="Istio APIs" category-value="istio-apis" >}}

Ingress ゲートウェイを設定します：

{{< text bash >}}
$ kubectl apply -f @samples/httpbin/httpbin-gateway.yaml@ -n foo
{{< /text >}}

[Ingress IP とポートの確認](/ja/docs/tasks/traffic-management/ingress/ingress-control/#ingress-ip-and-port-determination)の説明に従って、
`INGRESS_PORT` と `INGRESS_HOST` 環境変数を設定します。

{{< /tab >}}

{{< tab name="Gateway API" category-value="gateway-api" >}}

Ingress ゲートウェイを作成します：

{{< text bash >}}
$ kubectl apply -f @samples/httpbin/gateway-api/httpbin-gateway.yaml@ -n foo
$ kubectl wait --for=condition=programmed gtw -n foo httpbin-gateway
{{< /text >}}

`INGRESS_PORT` と `INGRESS_HOST` 環境変数を設定します：

{{< text bash >}}
$ export INGRESS_HOST=$(kubectl get gtw httpbin-gateway -n foo -o jsonpath='{.status.addresses[0].value}')
$ export INGRESS_PORT=$(kubectl get gtw httpbin-gateway -n foo -o jsonpath='{.spec.listeners[?(@.name=="http")].port}')
{{< /text >}}

{{< /tab >}}

{{< /tabset >}}

Ingress ゲートウェイを使用してテストクエリを実行します：

{{< text bash >}}
$ curl "$INGRESS_HOST:$INGRESS_PORT/headers" -s -o /dev/null -w "%{http_code}\n"
200
{{< /text >}}

これで、Ingress ゲートウェイがエンドユーザーの JWT を指定する認証ポリシーを追加します。

{{< tabset category-name="config-api" >}}

{{< tab name="Istio APIs" category-value="istio-apis" >}}

{{< text bash >}}
$ kubectl apply -f - <<EOF
apiVersion: security.istio.io/v1
kind: RequestAuthentication
metadata:
  name: "jwt-example"
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

{{< /tab >}}

{{< tab name="Gateway API" category-value="gateway-api" >}}

{{< text bash >}}
$ kubectl apply -f - <<EOF
apiVersion: security.istio.io/v1
kind: RequestAuthentication
metadata:
  name: "jwt-example"
  namespace: foo
spec:
  targetRef:
    kind: Gateway
    group: gateway.networking.k8s.io
    name: httpbin-gateway
  jwtRules:
  - issuer: "testing@secure.istio.io"
    jwksUri: "{{< github_file >}}/security/tools/jwt/samples/jwks.json"
EOF
{{< /text >}}

{{< /tab >}}

{{< /tabset >}}

選択されたワークロードの名前空間にポリシーを適用します。この例では、Ingress ゲートウェイです。

認証ヘッダーにトークンを提供し、その位置が暗黙的なデフォルトである場合、
Istio は[公開鍵セット]({{< github_file >}}/security/tools/jwt/samples/jwks.json)を使用してトークンを検証し、
無効なトークンのリクエストを拒否します。ただし、トークンなしのリクエストは受け入れられます。
この動作を観察するために、トークンなし、エラートークン、および有効なトークンを含むリクエストを再送信してみてください。

{{< text bash >}}
$ curl "$INGRESS_HOST:$INGRESS_PORT/headers" -s -o /dev/null -w "%{http_code}\n"
200
{{< /text >}}

{{< text bash >}}
$ curl --header "Authorization: Bearer deadbeef" "$INGRESS_HOST:$INGRESS_PORT/headers" -s -o /dev/null -w "%{http_code}\n"
401
{{< /text >}}

{{< text bash >}}
$ TOKEN=$(curl {{< github_file >}}/security/tools/jwt/samples/demo.jwt -s)
$ curl --header "Authorization: Bearer $TOKEN" "$INGRESS_HOST:$INGRESS_PORT/headers" -s -o /dev/null -w "%{http_code}\n"
200
{{< /text >}}

JWT 検証の他の側面を観察するために、スクリプト
[`gen-jwt.py`]({{< github_tree >}}/security/tools/jwt/samples/gen-jwt.py)
を使用して、異なる発行者、オーディエンス、有効期限などを持つ新しいトークンを生成してテストします。
Istio ライブラリからこのスクリプトをダウンロードできます：

{{< text bash >}}
$ wget --no-verbose {{< github_file >}}/security/tools/jwt/samples/gen-jwt.py
{{< /text >}}

`key.pem` ファイルも必要です：

{{< text bash >}}
$ wget --no-verbose {{< github_file >}}/security/tools/jwt/samples/key.pem
{{< /text >}}

{{< tip >}}
システムに `jwcrypto` ライブラリがインストールされていない場合は、
[jwcrypto](https://pypi.org/project/jwcrypto) からダウンロードしてインストールする必要があります。
{{< /tip >}}

JWT 認証には 60 秒のクロックシュークがあります。これは、JWT
トークンがその設定 `nbf` よりも 60 秒前に有効になり、その設定 `exp` よりも 60 秒後も有効であることを意味します。

例えば、以下のコマンドは 5 秒後に期限切れになるトークンを作成します。
Istio は、これらのトークンを 65 秒後に拒否するまで認証を続けま

{{< text bash >}}
$ TOKEN=$(python3 ./gen-jwt.py ./key.pem --expire 5)
$ for i in $(seq 1 10); do curl --header "Authorization: Bearer $TOKEN" "$INGRESS_HOST:$INGRESS_PORT/headers" -s -o /dev/null -w "%{http_code}\n"; curl 10; done
200
200
200
200
200
200
200
401
401
401
{{< /text >}}

`ingress gateway` に JWT ポリシーを追加することもできます（例えば、サービス
`istio-ingressgateway.istio-system.svc.cluster.local`）。
これは、このゲートウェイにバインドされたすべてのサービスに対して JWT ポリシーを定義するために使用されます。

### 有効なトークンを提供する {#require-a-valid-token}

有効なトークンを持たないリクエストを拒否するには、`DENY` 認証ポリシーを追加する必要があります。
以下の例の `notRequestPrincipals:["*"]` 設定を参照してください。
有効な JWT トークンを提供する場合にのみリクエスト主体が使用可能になるため、このルールは有効なトークンを持たないリクエストを拒否します。

{{< tabset category-name="config-api" >}}

{{< tab name="Istio APIs" category-value="istio-apis" >}}

{{< text bash >}}
$ kubectl apply -f - <<EOF
apiVersion: security.istio.io/v1
kind: AuthorizationPolicy
metadata:
  name: "frontend-ingress"
  namespace: istio-system
spec:
  selector:
    matchLabels:
      istio: ingressgateway
  action: DENY
  rules:
  - from:
    - source:
        notRequestPrincipals: ["*"]
EOF
{{< /text >}}

{{< /tab >}}

{{< tab name="Gateway API" category-value="gateway-api" >}}

{{< text bash >}}
$ kubectl apply -f - <<EOF
apiVersion: security.istio.io/v1
kind: AuthorizationPolicy
metadata:
  name: "frontend-ingress"
  namespace: foo
spec:
  targetRef:
    kind: Gateway
    group: gateway.networking.k8s.io
    name: httpbin-gateway
  action: DENY
  rules:
  - from:
    - source:
        notRequestPrincipals: ["*"]
EOF
{{< /text >}}

{{< /tab >}}

{{< /tabset >}}

トークンなしのリクエストを再送信します。リクエストが失敗し、エラーコード `403` が返されます：

{{< text bash >}}
$ curl "$INGRESS_HOST:$INGRESS_PORT/headers" -s -o /dev/null -w "%{http_code}\n"
403
{{< /text >}}

### パスごとに有効なトークンを提供する {#require-valid-tokens-per-path}

パスごとに有効なトークンを提供するには、そのパスを認可ポリシーで指定する必要があります。
例えば、以下の設定では `/headers` のみ JWT が必要です。認可ポリシーが有効になると、
`$INGRESS_HOST:$INGRESS_PORT/headers` へのリクエストは失敗し、エラーコード `403` が返されます。
他のすべてのパス（例えば `$INGRESS_HOST:$INGRESS_PORT/ip`）へのリクエストは成功します。

{{< tabset category-name="config-api" >}}

{{< tab name="Istio APIs" category-value="istio-apis" >}}

{{< text bash >}}
$ kubectl apply -f - <<EOF
apiVersion: security.istio.io/v1
kind: AuthorizationPolicy
metadata:
  name: "frontend-ingress"
  namespace: istio-system
spec:
  selector:
    matchLabels:
      istio: ingressgateway
  action: DENY
  rules:
  - from:
    - source:
        notRequestPrincipals: ["*"]
    to:
    - operation:
        paths: ["/headers"]
EOF
{{< /text >}}

{{< /tab >}}

{{< tab name="Gateway API" category-value="gateway-api" >}}

{{< text bash >}}
$ kubectl apply -f - <<EOF
apiVersion: security.istio.io/v1
kind: AuthorizationPolicy
metadata:
  name: "frontend-ingress"
  namespace: foo
spec:
  targetRef:
    kind: Gateway
    group: gateway.networking.k8s.io
    name: httpbin-gateway
  action: DENY
  rules:
  - from:
    - source:
        notRequestPrincipals: ["*"]
    to:
    - operation:
        paths: ["/headers"]
EOF
{{< /text >}}

{{< /tab >}}

{{< /tabset >}}

{{< text bash >}}
$ curl "$INGRESS_HOST:$INGRESS_PORT/headers" -s -o /dev/null -w "%{http_code}\n"
403
{{< /text >}}

{{< text bash >}}
$ curl "$INGRESS_HOST:$INGRESS_PORT/ip" -s -o /dev/null -w "%{http_code}\n"
200
{{< /text >}}

### 第 3 部のクリーンアップ {#cleanup-part-3}

1. 認証ポリシーを削除します：

    {{< text bash >}}
    $ kubectl -n istio-system delete requestauthentication jwt-example
    {{< /text >}}

1. 認可ポリシーを削除します：

    {{< text bash >}}
    $ kubectl -n istio-system delete authorizationpolicy frontend-ingress
    {{< /text >}}

1. トークン生成スクリプトとキーファイルを削除します：

    {{< text bash >}}
    $ rm -f ./gen-jwt.py ./key.pem
    {{< /text >}}

1. 後続の章のタスクを続行しない場合は、これらのテスト名前空間を削除するだけで、すべてのリソースを削除できます：

    {{< text bash >}}
    $ kubectl delete ns foo bar legacy
    {{< /text >}}
