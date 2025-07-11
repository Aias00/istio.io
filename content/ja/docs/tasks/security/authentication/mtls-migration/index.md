---
title: 双方向 TLS への移行
description: Istio サービスを段階的に双方向 TLS 通信モードへ移行する方法を説明します。
weight: 40
keywords: [security, authentication, migration]
aliases:
  - /zh/docs/tasks/security/mtls-migration/
owner: istio/wg-security-maintainers
test: yes
---

このタスクでは、Istio サービスのリクエストをプレーンテキストモードから双方向 TLS モードへスムーズに移行し、
移行中もオンラインのトラフィック通信に影響を与えない方法を説明します。

他のワークロードを呼び出す際、Istio はワークロードの Sidecar を自動的に
[双方向 TLS](/ja/docs/tasks/security/authentication/authn-policy/#auto-mutual-tls) を使うように設定します。
デフォルトでは、Istio はターゲットワークロードを `PERMISSIVE` モードで設定します。
`PERMISSIVE` モードが有効な場合、サービスはプレーンテキストと双方向 TLS の両方のトラフィックを受け入れます。
双方向 TLS トラフィックのみを許可したい場合は、設定を `STRICT` モードに変更する必要があります。

[Grafana ダッシュボード](/ja/docs/tasks/observability/metrics/using-istio-dashboard/) を使って、
どのサービスがまだ `PERMISSIVE` モードのサービスにプレーンテキストリクエストを送信しているかを確認し、
それらのサービスの移行が完了した後に、双方向 TLS リクエストのみを受け付けるようにロックダウンできます。

## 始める前に {#before-you-begin}

- Istio の[認証ポリシー](/ja/docs/concepts/security/#authentication-policies)および関連する[双方向 TLS 認証](/ja/docs/concepts/security/#mutual-tls-authentication)の概念を理解していること。

- [認証ポリシータスク](/ja/docs/tasks/security/authentication/authn-policy)を読み、
  認証ポリシーの設定方法を理解していること。

- Istio をインストールした Kubernetes クラスターを用意し、
  グローバルな双方向 TLS を有効にしていないこと（例：[インストール手順](/ja/docs/setup/getting-started)で説明されている `default` 設定ファイルを使用）。

このタスクでは、サンプルワークロードを作成し、ポリシーを変更してワークロード間で
STRICT 双方向 TLS を強制することで、移行プロセスを試すことができます。

## クラスターのセットアップ {#set-up-the-cluster}

- 2 つの名前空間 `foo` と `bar` を作成し、両方の名前空間に
  [httpbin]({{< github_tree >}}/samples/httpbin)、
  [curl]({{< github_tree >}}/samples/curl)、および Sidecar をデプロイします。

  {{< text bash >}}
  $ kubectl create ns foo
  $ kubectl apply -f <(istioctl kube-inject -f @samples/httpbin/httpbin.yaml@) -n foo
  $ kubectl apply -f <(istioctl kube-inject -f @samples/curl/curl.yaml@) -n foo
  $ kubectl create ns bar
  $ kubectl apply -f <(istioctl kube-inject -f @samples/httpbin/httpbin.yaml@) -n bar
  $ kubectl apply -f <(istioctl kube-inject -f @samples/curl/curl.yaml@) -n bar
  {{< /text >}}

- もう 1 つの名前空間 `legacy` を作成し、Sidecar を注入せずに
  [curl]({{< github_tree >}}/samples/curl) をデプロイします：

  {{< text bash >}}
  $ kubectl create ns legacy
  $ kubectl apply -f @samples/curl/curl.yaml@ -n legacy
  {{< /text >}}

- 各 curl Pod（`foo`、`bar`、`legacy` の各名前空間）から
  `httpbin.foo` へ http リクエストを送信します。すべてのリクエストは正常に応答し、HTTP code 200 を返すはずです。

  {{< text bash >}}
  $ for from in "foo" "bar" "legacy"; do for to in "foo" "bar"; do kubectl exec "$(kubectl get pod -l app=curl -n ${from} -o jsonpath={.items..metadata.name})" -c curl -n ${from} -- curl http://httpbin.${to}:8000/ip -s -o /dev/null -w "curl.${from} to httpbin.${to}: %{http_code}\n"; done; done
  curl.foo to httpbin.foo: 200
  curl.foo to httpbin.bar: 200
  curl.bar to httpbin.foo: 200
  curl.bar to httpbin.bar: 200
  curl.legacy to httpbin.foo: 200
  curl.legacy to httpbin.bar: 200
  {{< /text >}}

  {{< tip >}}

  もし curl コマンドが失敗した場合は、httpbin サービスリクエストに影響する
  既存の認証ポリシーや DestinationRule がないか確認してください。

  {{< text bash >}}
  $ kubectl get peerauthentication --all-namespaces
  No resources found
  {{< /text >}}

  {{< text bash >}}
  $ kubectl get destinationrule --all-namespaces
  No resources found
  {{< /text >}}

  {{< /tip >}}

## 名前空間単位で双方向 TLS へロックダウン {#lock-down-to-mutual-tls-by-namespace}

すべてのクライアントサービスが Istio へ移行し、Envoy Sidecar を注入した後、
`httpbin.foo` を双方向 TLS リクエストのみ受け付けるようにロックダウンできます。

{{< text bash >}}
$ kubectl apply -n foo -f - <<EOF
apiVersion: security.istio.io/v1
kind: PeerAuthentication
metadata:
name: default
spec:
mtls:
mode: STRICT
EOF
{{< /text >}}

この時点で、`curl.legacy` からのリクエストは失敗します。

{{< text bash >}}
$ for from in "foo" "bar" "legacy"; do for to in "foo" "bar"; do kubectl exec "$(kubectl get pod -l app=curl -n ${from} -o jsonpath={.items..metadata.name})" -c curl -n ${from} -- curl http://httpbin.${to}:8000/ip -s -o /dev/null -w "curl.${from} to httpbin.${to}: %{http_code}\n"; done; done
curl.foo to httpbin.foo: 200
curl.foo to httpbin.bar: 200
curl.bar to httpbin.foo: 200
curl.bar to httpbin.bar: 200
curl.legacy to httpbin.foo: 000
command terminated with exit code 56
curl.legacy to httpbin.bar: 200
{{< /text >}}

Istio を `values.global.proxy.privileged=true` でインストールしている場合は、
`tcpdump` を使ってトラフィックが暗号化されているか確認できます。

{{< text bash >}}
$ kubectl exec -nfoo "$(kubectl get pod -nfoo -lapp=httpbin -ojsonpath={.items..metadata.name})" -c istio-proxy -- sudo tcpdump dst port 80 -A
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on eth0, link-type EN10MB (Ethernet), capture size 262144 bytes
{{< /text >}}

`curl.legacy` と `curl.foo` からリクエストを送信すると、
出力にプレーンテキストと暗号化テキストの両方が表示されます。

すべてのサービスを Istio（Envoy Sidecar 注入）へ移行できない場合は、`PERMISSIVE` モードを有効にする必要があります。
ただし、`PERMISSIVE` モードではプレーンテキストリクエストに対して認証や認可チェックは行われません。
異なるリクエストパスごとに異なる認可ポリシーを設定するには、[Istio 認可](/ja/docs/tasks/security/authorization/authz-http/)の利用を推奨します。

## メッシュ全体で mTLS をロックダウン {#lock-down-mutual-TLS-for-the-entire-mesh}

{{< text bash >}}
$ kubectl apply -n istio-system -f - <<EOF
apiVersion: security.istio.io/v1
kind: PeerAuthentication
metadata:
name: default
spec:
mtls:
mode: STRICT
EOF
{{< /text >}}

これで、`foo` と `bar` の両方の名前空間で双方向 TLS トラフィックのみが強制されるため、
`curl.legacy` から両方の名前空間のサービスへのリクエストは失敗するはずです。

{{< text bash >}}
$ for from in "foo" "bar" "legacy"; do for to in "foo" "bar"; do kubectl exec "$(kubectl get pod -l app=curl -n ${from} -o jsonpath={.items..metadata.name})" -c curl -n ${from} -- curl http://httpbin.${to}:8000/ip -s -o /dev/null -w "curl.${from} to httpbin.${to}: %{http_code}\n"; done; done
{{< /text >}}

## クリーンアップ {#clean-up-the-example}

1. メッシュ全体の認証ポリシーを削除します。

   {{< text bash >}}
   $ kubectl delete peerauthentication -n foo default
   $ kubectl delete peerauthentication -n istio-system default
   {{< /text >}}

1. テスト用に作成した名前空間を削除します。

   {{< text bash >}}
   $ kubectl delete ns foo bar legacy
   Namespaces foo bar legacy deleted.
   {{< /text >}}
