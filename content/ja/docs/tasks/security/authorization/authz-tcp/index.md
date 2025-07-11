---
title: TCP トラフィック
description: TCP トラフィックのアクセス制御の設定方法を紹介します。
weight: 20
keywords: [security, access-control, rbac, tcp, authorization]
aliases:
  - /zh/docs/tasks/security/authz-tcp/
owner: istio/wg-security-maintainers
test: no
---

このタスクでは、Istio メッシュ内で TCP トラフィックに対して Istio 認可ポリシーを設定する方法を紹介します。

## 始める前に {#before-you-begin}

このタスクを始める前に、以下を完了してください：

- [Istio 認可の概念](/ja/docs/concepts/security/#authorization)を読んでください。

- [Istio インストールガイド](/ja/docs/setup/install/istioctl/)に従って Istio をインストールしてください。

- 名前空間（例：`foo`）に 2 つのワークロード、`curl` と `tcp-echo` をデプロイします。
  どちらのワークロードも前段に Envoy プロキシを持ちます。
  `tcp-echo` ワークロードは 9000、9001、9002 ポートでリッスンし、受信したすべてのトラフィックに `hello` を付けて返します。
  例えば、`world` を `tcp-echo` に送信すると、`hello world` と応答します。
  `tcp-echo` の Kubernetes Service オブジェクトは 9000 と 9001 ポートのみを宣言し、9002 ポートは省略されています。
  パススルーフィルターチェーンが 9002 ポートのトラフィックを処理します。以下のコマンドでサンプルの名前空間とワークロードをデプロイします：

  {{< text bash >}}
  $ kubectl create ns foo
  $ kubectl apply -f <(istioctl kube-inject -f @samples/tcp-echo/tcp-echo.yaml@) -n foo
  $ kubectl apply -f <(istioctl kube-inject -f @samples/curl/curl.yaml@) -n foo
  {{< /text >}}

- 次のコマンドで `curl` から `tcp-echo` の 9000 および 9001 ポートへの通信が正常であることを確認します：

  {{< text bash >}}
  $ kubectl exec "$(kubectl get pod -l app=curl -n foo -o jsonpath={.items..metadata.name})" \
   -c curl -n foo -- sh -c \
   'echo "port 9000" | nc tcp-echo 9000' | grep "hello" && echo 'connection succeeded' || echo 'connection rejected'
  hello port 9000
  connection succeeded
  {{< /text >}}

  {{< text bash >}}
  $ kubectl exec "$(kubectl get pod -l app=curl -n foo -o jsonpath={.items..metadata.name})" \
   -c curl -n foo -- sh -c \
   'echo "port 9001" | nc tcp-echo 9001' | grep "hello" && echo 'connection succeeded' || echo 'connection rejected'
  hello port 9001
  connection succeeded
  {{< /text >}}

- `curl` から `tcp-echo` の 9002 ポートへの通信が正常であることを確認します。
  9002 ポートは `tcp-echo` の Kubernetes Service オブジェクトで定義されていないため、トラフィックを直接 Pod IP に送信する必要があります。
  Pod IP アドレスを取得し、以下のコマンドでリクエストを送信します：

  {{< text bash >}}
  $ TCP_ECHO_IP=$(kubectl get pod "$(kubectl get pod -l app=tcp-echo -n foo -o jsonpath={.items..metadata.name})" -n foo -o jsonpath="{.status.podIP}")
  $ kubectl exec "$(kubectl get pod -l app=curl -n foo -o jsonpath={.items..metadata.name})" \
   -c curl -n foo -- sh -c \
   "echo \"port 9002\" | nc $TCP_ECHO_IP 9002" | grep "hello" && echo 'connection succeeded' || echo 'connection rejected'
  hello port 9002
  connection succeeded
  {{< /text >}}

{{< warning >}}
期待される出力が表示されない場合は、数秒後に再試行してください。キャッシュやその他の伝播遅延が原因となる場合があります。
{{< /warning >}}

## TCP ワークロードに ALLOW 認可ポリシーを設定する {#configure-allow-authorization-policy-for-a-tcp-workload}

1. `foo` 名前空間の `tcp-echo` ワークロードに `tcp-policy` 認可ポリシーを作成します。
   次のコマンドで 9000 および 9001 ポートへのリクエストを許可するポリシーを適用します：

   {{< text bash >}}
   $ kubectl apply -f - <<EOF
   apiVersion: security.istio.io/v1
   kind: AuthorizationPolicy
   metadata:
   name: tcp-policy
   namespace: foo
   spec:
   selector:
   matchLabels:
   app: tcp-echo
   action: ALLOW
   rules:

   - to: - operation:
     ports: ["9000", "9001"]
     EOF
     {{< /text >}}

1. 次のコマンドで 9000 ポートへのリクエストが許可されることを確認します：

   {{< text bash >}}
   $ kubectl exec "$(kubectl get pod -l app=curl -n foo -o jsonpath={.items..metadata.name})" \
    -c curl -n foo -- sh -c \
    'echo "port 9000" | nc tcp-echo 9000' | grep "hello" && echo 'connection succeeded' || echo 'connection rejected'
   hello port 9000
   connection succeeded
   {{< /text >}}

1. 次のコマンドで 9001 ポートへのリクエストが許可されることを確認します：

   {{< text bash >}}
   $ kubectl exec "$(kubectl get pod -l app=curl -n foo -o jsonpath={.items..metadata.name})" \
    -c curl -n foo -- sh -c \
    'echo "port 9001" | nc tcp-echo 9001' | grep "hello" && echo 'connection succeeded' || echo 'connection rejected'
   hello port 9001
   connection succeeded
   {{< /text >}}

1. 9002 ポートへのリクエストが拒否されることを確認します。`tcp-echo` の Kubernetes Service オブジェクトで明示的に宣言されていないポートでも、
   認可ポリシーはパススルーフィルターチェーンにも適用されます。次のコマンドを実行し、出力を確認します：

   {{< text bash >}}
   $ kubectl exec "$(kubectl get pod -l app=curl -n foo -o jsonpath={.items..metadata.name})" \
    -c curl -n foo -- sh -c \
    "echo \"port 9002\" | nc $TCP_ECHO_IP 9002" | grep "hello" && echo 'connection succeeded' || echo 'connection rejected'
   connection rejected
   {{< /text >}}

1. 次のコマンドで 9000 ポートに HTTP 専用フィールド `methods` を追加してポリシーを更新します：

   {{< text bash >}}
   $ kubectl apply -f - <<EOF
   apiVersion: security.istio.io/v1
   kind: AuthorizationPolicy
   metadata:
   name: tcp-policy
   namespace: foo
   spec:
   selector:
   matchLabels:
   app: tcp-echo
   action: ALLOW
   rules:

   - to: - operation:
     methods: ["GET"]
     ports: ["9000"]
     EOF
     {{< /text >}}

1. 9000 ポートへのリクエストが拒否されることを確認します。これは TCP トラフィックに HTTP 専用フィールド（`methods`）を使用したため、
   ルールが無効となり、Istio は無効な ALLOW ルールを無視します。結果として、リクエストはどの ALLOW ルールにも一致せず拒否されます。
   次のコマンドを実行し、出力を確認します：

   {{< text bash >}}
   $ kubectl exec "$(kubectl get pod -l app=curl -n foo -o jsonpath={.items..metadata.name})" \
    -c curl -n foo -- sh -c \
    'echo "port 9000" | nc tcp-echo 9000' | grep "hello" && echo 'connection succeeded' || echo 'connection rejected'
   connection rejected
   {{< /text >}}

1. 9001 ポートへのリクエストが拒否されることを確認します。これはリクエストがどの ALLOW ルールにも一致しないためです。次のコマンドを実行し、出力を確認します：

   {{< text bash >}}
   $ kubectl exec "$(kubectl get pod -l app=curl -n foo -o jsonpath={.items..metadata.name})" \
    -c curl -n foo -- sh -c \
    'echo "port 9001" | nc tcp-echo 9001' | grep "hello" && echo 'connection succeeded' || echo 'connection rejected'
   connection rejected
   {{< /text >}}

## TCP ワークロードに DENY 認可ポリシーを設定する {#configure-deny-authorization-policy-for-a-tcp-workload}

1. 次のコマンドで HTTP 専用フィールドを持つ DENY ポリシーを追加します：

   {{< text bash >}}
   $ kubectl apply -f - <<EOF
   apiVersion: security.istio.io/v1
   kind: AuthorizationPolicy
   metadata:
   name: tcp-policy
   namespace: foo
   spec:
   selector:
   matchLabels:
   app: tcp-echo
   action: DENY
   rules:

   - to: - operation:
     methods: ["GET"]
     EOF
     {{< /text >}}

1. 9000 ポートへのリクエストが拒否されることを確認します。これは Istio が tcp ポートに対する DENY ルールで HTTP 専用フィールドを理解せず、
   このルールが制限的な性質を持つため、tcp ポートへのすべてのトラフィックが拒否されるためです：

   {{< text bash >}}
   $ kubectl exec "$(kubectl get pod -l app=curl -n foo -o jsonpath={.items..metadata.name})" \
    -c curl -n foo -- sh -c \
    'echo "port 9000" | nc tcp-echo 9000' | grep "hello" && echo 'connection succeeded' || echo 'connection rejected'
   connection rejected
   {{< /text >}}

1. 9001 ポートへのリクエストが拒否されることを確認します。理由は上記と同じです。

   {{< text bash >}}
   $ kubectl exec "$(kubectl get pod -l app=curl -n foo -o jsonpath={.items..metadata.name})" \
    -c curl -n foo -- sh -c \
    'echo "port 9001" | nc tcp-echo 9001' | grep "hello" && echo 'connection succeeded' || echo 'connection rejected'
   connection rejected
   {{< /text >}}

1. 次のコマンドで TCP と HTTP フィールドの両方を持つ DENY ポリシーを追加します：

   {{< text bash >}}
   $ kubectl apply -f - <<EOF
   apiVersion: security.istio.io/v1
   kind: AuthorizationPolicy
   metadata:
   name: tcp-policy
   namespace: foo
   spec:
   selector:
   matchLabels:
   app: tcp-echo
   action: DENY
   rules:

   - to: - operation:
     methods: ["GET"]
     ports: ["9000"]
     EOF
     {{< /text >}}

1. 9000 ポートへのリクエストが拒否されることを確認します。これはリクエストが上記の DENY ポリシーの `ports` に一致するためです：

   {{< text bash >}}
   $ kubectl exec "$(kubectl get pod -l app=curl -n foo -o jsonpath={.items..metadata.name})" \
    -c curl -n foo -- sh -c \
    'echo "port 9000" | nc tcp-echo 9000' | grep "hello" && echo 'connection succeeded' || echo 'connection rejected'
   connection rejected
   {{< /text >}}

1. 9001 ポートへのリクエストが許可されることを確認します。これはリクエストが DENY ポリシーの `ports` に一致しないためです：

   {{< text bash >}}
   $ kubectl exec "$(kubectl get pod -l app=curl -n foo -o jsonpath={.items..metadata.name})" \
    -c curl -n foo -- sh -c \
    'echo "port 9001" | nc tcp-echo 9001' | grep "hello" && echo 'connection succeeded' || echo 'connection rejected'
   hello port 9001
   connection succeeded
   {{< /text >}}

## クリーンアップ {#cleanup}

`foo` 名前空間を削除します：

{{< text bash >}}
$ kubectl delete namespace foo
{{< /text >}}
