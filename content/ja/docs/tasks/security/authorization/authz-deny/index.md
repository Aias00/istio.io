---
title: 明示的拒否
description: アクセス制御でトラフィックを明示的に拒否する方法。
weight: 40
keywords: [security, access-control, rbac, authorization, deny]
owner: istio/wg-security-maintainers
test: yes
---

このタスクでは、`DENY` アクションを使った Istio 認可ポリシーで、Istio メッシュ内のトラフィックを明示的に拒否する方法を紹介します。
これは `ALLOW` アクションとは異なり、`DENY` アクションはより高い優先度を持ち、どの `ALLOW` アクションでも上書きされません。

## 始める前に {#before-you-begin}

始める前に、以下を実施してください：

- [Istio 認可の概念](/ja/docs/concepts/security/#authorization)を読んでください。

- [Istio インストールガイド](/ja/docs/setup/install/istioctl/)に従って Istio をインストールしてください。

- ワークロードをデプロイします：

  このタスクでは、`foo` 名前空間に `httpbin` と `curl` の 2 つのワークロードを使用します。
  どちらも Envoy プロキシを実行しています。以下のコマンドでサンプル名前空間とワークロードをデプロイします：

  {{< text bash >}}
  $ kubectl create ns foo
  $ kubectl apply -f <(istioctl kube-inject -f @samples/httpbin/httpbin.yaml@) -n foo
  $ kubectl apply -f <(istioctl kube-inject -f @samples/curl/curl.yaml@) -n foo
  {{< /text >}}

- 次のコマンドで `curl` から `httpbin` への通信を確認します。

  {{< text bash >}}
  $ kubectl exec "$(kubectl get pod -l app=curl -n foo -o jsonpath={.items..metadata.name})" -c curl -n foo -- curl http://httpbin.foo:8000/ip -sS -o /dev/null -w "%{http_code}\n"
  200
  {{< /text >}}

{{< warning >}}
期待される出力が表示されない場合は、数秒後に再試行してください。
キャッシュや伝播の遅延が原因となる場合があります。
{{< /warning >}}

## リクエストを明示的に拒否する {#explicitly-deny-a-request}

1. 次のコマンドで、`foo` 名前空間の `httpbin` ワークロードに `deny-method-get` 認可ポリシーを作成します。
   この認可ポリシーは `action` を `DENY` に設定し、`rules` セクションで指定した条件に一致するリクエストを拒否します。
   このタイプのポリシーは「拒否ポリシー」と呼ばれます。この例では、リクエストメソッドが `GET` の場合にリクエストを拒否します。

   {{< text bash >}}
   $ kubectl apply -f - <<EOF
   apiVersion: security.istio.io/v1
   kind: AuthorizationPolicy
   metadata:
   name: deny-method-get
   namespace: foo
   spec:
   selector:
   matchLabels:
   app: httpbin
   action: DENY
   rules:

   - to: - operation:
     methods: ["GET"]
     EOF
     {{< /text >}}

1. `GET` リクエストが拒否されることを確認します：

   {{< text bash >}}
   $ kubectl exec "$(kubectl get pod -l app=curl -n foo -o jsonpath={.items..metadata.name})" -c curl -n foo -- curl "http://httpbin.foo:8000/get" -X GET -sS -o /dev/null -w "%{http_code}\n"
   403
   {{< /text >}}

1. `POST` リクエストが許可されることを確認します：

   {{< text bash >}}
   $ kubectl exec "$(kubectl get pod -l app=curl -n foo -o jsonpath={.items..metadata.name})" -c curl -n foo -- curl "http://httpbin.foo:8000/post" -X POST -sS -o /dev/null -w "%{http_code}\n"
   200
   {{< /text >}}

1. `deny-method-get` 認可ポリシーを更新し、HTTP ヘッダー `x-token` の値が `admin` でない場合のみ `GET` リクエストを拒否するようにします。以下のポリシー例では、`notValues` フィールドを `["admin"]` に設定し、HTTP ヘッダー `x-token` の値が `admin` 以外のリクエストを拒否します：

   {{< text bash >}}
   $ kubectl apply -f - <<EOF
   apiVersion: security.istio.io/v1
   kind: AuthorizationPolicy
   metadata:
   name: deny-method-get
   namespace: foo
   spec:
   selector:
   matchLabels:
   app: httpbin
   action: DENY
   rules:

   - to: - operation:
     methods: ["GET"]
     when: - key: request.headers[x-token]
     notValues: ["admin"]
     EOF
     {{< /text >}}

1. HTTP ヘッダー `x-token: admin` 付きの `GET` リクエストが許可されることを確認します：

   {{< text bash >}}
   $ kubectl exec "$(kubectl get pod -l app=curl -n foo -o jsonpath={.items..metadata.name})" -c curl -n foo -- curl "http://httpbin.foo:8000/get" -X GET -H "x-token: admin" -sS -o /dev/null -w "%{http_code}\n"
   200
   {{< /text >}}

1. HTTP ヘッダー `x-token: guest` 付きの GET リクエストが拒否されることを確認します：

   {{< text bash >}}
   $ kubectl exec "$(kubectl get pod -l app=curl -n foo -o jsonpath={.items..metadata.name})" -c curl -n foo -- curl "http://httpbin.foo:8000/get" -X GET -H "x-token: guest" -sS -o /dev/null -w "%{http_code}\n"
   403
   {{< /text >}}

1. 次のコマンドで `allow-path-ip` 認可ポリシーを作成し、`/ip` パスへのリクエストを `httpbin` ワークロードで許可します。この認可ポリシーは `action` フィールドを `ALLOW` に設定し、このタイプのポリシーは「許可ポリシー」と呼ばれます。

   {{< text bash >}}
   $ kubectl apply -f - <<EOF
   apiVersion: security.istio.io/v1
   kind: AuthorizationPolicy
   metadata:
   name: allow-path-ip
   namespace: foo
   spec:
   selector:
   matchLabels:
   app: httpbin
   action: ALLOW
   rules:

   - to: - operation:
     paths: ["/ip"]
     EOF
     {{< /text >}}

1. `/ip` パスへの HTTP ヘッダー `x-token: guest` 付きの `GET` リクエストが `deny-method-get` ポリシーで拒否されることを確認します。「拒否ポリシー」は「許可ポリシー」よりも優先されます：

   {{< text bash >}}
   $ kubectl exec "$(kubectl get pod -l app=curl -n foo -o jsonpath={.items..metadata.name})" -c curl -n foo -- curl "http://httpbin.foo:8000/ip" -X GET -H "x-token: guest" -s -o /dev/null -w "%{http_code}\n"
   403
   {{< /text >}}

1. `/ip` パスへの HTTP ヘッダー `x-token: admin` 付きの `GET` リクエストが `allow-path-ip` ポリシーで許可されることを確認します：

   {{< text bash >}}
   $ kubectl exec "$(kubectl get pod -l app=curl -n foo -o jsonpath={.items..metadata.name})" -c curl -n foo -- curl "http://httpbin.foo:8000/ip" -X GET -H "x-token: admin" -s -o /dev/null -w "%{http_code}\n"
   200
   {{< /text >}}

1. `/get` パスへの HTTP ヘッダー `x-token: admin` 付きの `GET` リクエストが拒否されることを確認します。
   これは `allow-path-ip` ポリシーに一致しないためです：

   {{< text bash >}}
   $ kubectl exec "$(kubectl get pod -l app=curl -n foo -o jsonpath={.items..metadata.name})" -c curl -n foo -- curl "http://httpbin.foo:8000/get" -X GET -H "x-token: admin" -s -o /dev/null -w "%{http_code}\n"
   403
   {{< /text >}}

## クリーンアップ {#clean-up}

foo 名前空間を削除します：

{{< text bash >}}
$ kubectl delete namespace foo
{{< /text >}}
