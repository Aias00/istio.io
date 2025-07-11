---
title: 外部認可
description: アクセス制御を外部認可システムに委譲する方法。
weight: 35
keywords:
  [
    security,
    access-control,
    rbac,
    authorization,
    custom,
    opa,
    oauth,
    oauth2-proxy,
  ]
owner: istio/wg-security-maintainers
test: yes
---

このタスクでは、新しい [action](/ja/docs/reference/config/security/authorization-policy/#AuthorizationPolicy-Action)
フィールド `CUSTOM` を使い、Istio 認可ポリシーでアクセス制御を外部認可システムに委譲する方法を紹介します。これにより、
[OPA authorization](https://www.openpolicyagent.org/docs/envoy)、
[`oauth2-proxy`](https://github.com/oauth2-proxy/oauth2-proxy) または独自の外部認可サーバーと統合できます。

## 始める前に {#before-you-begin}

始める前に、以下を実施してください：

- [Istio 認可の概念](/ja/docs/concepts/security/#authorization)を読んでください。

- [Istio インストールガイド](/ja/docs/setup/install/istioctl/)に従って Istio をインストールしてください。

- テスト用ワークロードをデプロイします：

  このタスクでは、`foo` 名前空間に `httpbin` と `curl` の 2 つのワークロードを使用します。
  どちらも Envoy サイドカーコンテナを含みます。以下のコマンドでサンプル名前空間とワークロードをデプロイします：

  {{< text bash >}}
  $ kubectl create ns foo
  $ kubectl label ns foo istio-injection=enabled
  $ kubectl apply -f @samples/httpbin/httpbin.yaml@ -n foo
  $ kubectl apply -f @samples/curl/curl.yaml@ -n foo
  {{< /text >}}

- 次のコマンドで `curl` から `httpbin` へのアクセスを確認します：

  {{< text bash >}}
  $ kubectl exec "$(kubectl get pod -l app=curl -n foo -o jsonpath={.items..metadata.name})" -c curl -n foo -- curl http://httpbin.foo:8000/ip -s -o /dev/null -w "%{http_code}\n"
  200
  {{< /text >}}

{{< warning >}}
期待される出力が表示されない場合は、数秒後に再試行してください。キャッシュや伝播の遅延が原因となる場合があります。
{{< /warning >}}

## 外部認可サーバーのデプロイ {#deploy-the-external-authorizer}

まず、外部認可サーバーをデプロイする必要があります。サンプル外部認可サーバーをメッシュ内の独立した Pod としてデプロイします。

1. 次のコマンドでサンプル外部認可サーバーをデプロイします：

   {{< text bash >}}
   $ kubectl apply -n foo -f {{< github_file >}}/samples/extauthz/ext-authz.yaml
   service/ext-authz created
   deployment.apps/ext-authz created
   {{< /text >}}

1. サンプル外部認可サーバーが起動していることを確認します：

   {{< text bash >}}
   $ kubectl logs "$(kubectl get pod -l app=ext-authz -n foo -o jsonpath={.items..metadata.name})" -n foo -c ext-authz
   2021/01/07 22:55:47 Starting HTTP server at [::]:8000
   2021/01/07 22:55:47 Starting gRPC server at [::]:9000
   {{< /text >}}

また、外部認可サーバーを外部認可が必要なアプリケーションと同じ Pod 内にデプロイしたり、
メッシュ外にデプロイすることもできます。その場合は ServiceEntry リソースを作成してサービスをメッシュに登録し、
プロキシがアクセスできるようにしてください。

以下は、外部認可サーバーをアプリケーションと同じ Pod 内にデプロイする場合の ServiceEntry の例です。

{{< text yaml >}}
apiVersion: networking.istio.io/v1
kind: ServiceEntry
metadata:
name: external-authz-grpc-local
spec:
hosts:

- "external-authz-grpc.local" # メッシュ設定の拡張プロバイダーで使うサービス名
  endpoints:
- address: "127.0.0.1"
  ports:
- name: grpc
  number: 9191 # 拡張プロバイダーで使うポート番号
  protocol: GRPC
  resolution: STATIC
  {{< /text >}}

## 外部認可プロバイダーの定義 {#define-the-external-authorizer}

認可ポリシーで `CUSTOM` アクションを使うには、メッシュで利用可能な外部認可プロバイダーを定義する必要があります。
これは現在、メッシュ設定の[拡張プロバイダー](https://github.com/istio/api/blob/a205c627e4b955302bbb77dd837c8548e89e6e64/mesh/v1alpha1/config.proto#L534)で定義します。

現時点で唯一サポートされている拡張プロバイダータイプは [Envoy `ext_authz`](https://www.envoyproxy.io/docs/envoy/v1.16.2/intro/arch_overview/security/ext_authz_filter) です。
外部認可サーバーは対応する Envoy `ext_authz` チェックインターフェースを実装する必要があります。

このタスクでは、リクエストヘッダー `x-ext-authz: allow` がある場合に許可する
[サンプル外部認可サーバー]({{< github_tree >}}/samples/extauthz)を使用します。

1. 次のコマンドでメッシュ設定を編集します：

   {{< text bash >}}
   $ kubectl edit configmap istio -n istio-system
   {{< /text >}}

1. エディタで、以下のように拡張プロバイダー定義を追加します：

   以下は、Service `ext-authz.foo.svc.cluster.local` を使う 2 つの外部プロバイダー
   `sample-ext-authz-grpc` と `sample-ext-authz-http` の定義例です。このサービスは Envoy `ext_authz`
   フィルターで定義された HTTP および GRPC チェック API を実装しています。次のステップでこのサービスをデプロイします。

   {{< text yaml >}}
   data:
   mesh: |- # 以下を追加して外部認可プロバイダーを定義します。
   extensionProviders: - name: "sample-ext-authz-grpc"
   envoyExtAuthzGrpc:
   service: "ext-authz.foo.svc.cluster.local"
   port: "9000" - name: "sample-ext-authz-http"
   envoyExtAuthzHttp:
   service: "ext-authz.foo.svc.cluster.local"
   port: "8000"
   includeRequestHeadersInCheck: ["x-ext-authz"]
   {{< /text >}}

   また、拡張プロバイダーを調整して EXT_AUTHZ フィルターの動作を制御できます。
   例えば、どのヘッダーを外部認可サーバーやアプリケーションバックエンドに送るか、エラー時のステータスなどです。

   例えば、[`oauth2-proxy`](https://github.com/oauth2-proxy/oauth2-proxy) と連携する拡張プロバイダーの例：

   {{< text yaml >}}
   data:
   mesh: |-
   extensionProviders: - name: "oauth2-proxy"
   envoyExtAuthzHttp:
   service: "oauth2-proxy.foo.svc.cluster.local"
   port: "4180" # oauth2-proxy のデフォルトポート
   includeRequestHeadersInCheck: ["authorization", "cookie"] # oauth2-proxy へ送るリクエストヘッダー
   headersToUpstreamOnAllow: ["authorization", "path", "x-auth-request-user", "x-auth-request-email", "x-auth-request-access-token"] # 許可時にバックエンドへ送るヘッダー
   headersToDownstreamOnAllow: ["set-cookie"] # 許可時にクライアントへ返すヘッダー
   headersToDownstreamOnDeny: ["content-type", "set-cookie"] # 拒否時にクライアントへ返すヘッダー
   {{< /text >}}

## 外部認可の有効化 {#enable-with-external-authorization}

これで外部認可サーバーが認可ポリシーで利用できるようになりました。

1. 次のコマンドで外部認可を有効化します：

   以下のコマンドは、`CUSTOM` アクション値を持つ認可ポリシーを `httpbin` ワークロードに適用します。
   このポリシーは、`sample-ext-authz-grpc` で定義した外部認可サーバーを使い、パス `/headers` へのリクエストで外部認可を行います。

   {{< text bash >}}
   $ kubectl apply -n foo -f - <<EOF
   apiVersion: security.istio.io/v1
   kind: AuthorizationPolicy
   metadata:
   name: ext-authz
   spec:
   selector:
   matchLabels:
   app: httpbin
   action: CUSTOM
   provider: # プロバイダー名は MeshConfig で定義した拡張プロバイダーと一致する必要があります # sample-ext-authz-http に変更して別の外部認可サーバーもテストできます
   name: sample-ext-authz-grpc
   rules:

   # rules で外部認可サーバーを呼び出す条件を指定します

   - to: - operation:
     paths: ["/headers"]
     EOF
     {{< /text >}}

   実行時、`httpbin` ワークロードの `/headers` パスへのリクエストは `ext_authz` フィルターで一時停止され、
   外部認可サーバーへチェックリクエストが送信され、許可・拒否が決定されます。

1. `x-ext-authz: deny` ヘッダー付きで `/headers` パスへリクエストした場合、`ext_authz` サンプルサーバーが拒否することを確認します：

   {{< text bash >}}
   $ kubectl exec "$(kubectl get pod -l app=curl -n foo -o jsonpath={.items..metadata.name})" -c curl -n foo -- curl "http://httpbin.foo:8000/headers" -H "x-ext-authz: deny" -s
   denied by ext_authz for not found header `x-ext-authz: allow` in the request
   {{< /text >}}

1. `x-ext-authz: allow` ヘッダー付きで `/headers` パスへリクエストした場合、許可されることを確認します：

   {{< text bash >}}
   $ kubectl exec "$(kubectl get pod -l app=curl -n foo -o jsonpath={.items..metadata.name})" -c curl -n foo -- curl "http://httpbin.foo:8000/headers" -H "x-ext-authz: allow" -s | jq '.headers'
   ...
   "X-Ext-Authz-Check-Result": [
   "allowed"
   ],
   ...
   {{< /text >}}

1. `/ip` パスへのリクエストは許可され、外部認可はトリガーされないことを確認します：

   {{< text bash >}}
   $ kubectl exec "$(kubectl get pod -l app=curl -n foo -o jsonpath={.items..metadata.name})" -c curl -n foo -- curl "http://httpbin.foo:8000/ip" -s -o /dev/null -w "%{http_code}\n"
   200
   {{< /text >}}

1. `ext_authz` サンプルサーバーのログを確認し、2 回呼び出されていること（1 回は許可、1 回は拒否）を確認します：

   {{< text bash >}}
   $ kubectl logs "$(kubectl get pod -l app=ext-authz -n foo -o jsonpath={.items..metadata.name})" -n foo -c ext-authz
   2021/01/07 22:55:47 Starting HTTP server at [::]:8000
   2021/01/07 22:55:47 Starting gRPC server at [::]:9000
   2021/01/08 03:25:00 [gRPCv3][denied]: httpbin.foo:8000/headers, attributes: source:{address:{socket_address:{address:"10.44.0.22" port_value:52088}} principal:"spiffe://cluster.local/ns/foo/sa/curl"} destination:{address:{socket_address:{address:"10.44.3.30" port_value:80}} principal:"spiffe://cluster.local/ns/foo/sa/httpbin"} request:{time:{seconds:1610076306 nanos:473835000} http:{id:"13869142855783664817" method:"GET" headers:{key:":authority" value:"httpbin.foo:8000"} headers:{key:":method" value:"GET"} headers:{key:":path" value:"/headers"} headers:{key:"accept" value:"_/_"} headers:{key:"content-length" value:"0"} headers:{key:"user-agent" value:"curl/7.74.0-DEV"} headers:{key:"x-b3-sampled" value:"1"} headers:{key:"x-b3-spanid" value:"377ba0cdc2334270"} headers:{key:"x-b3-traceid" value:"635187cb20d92f62377ba0cdc2334270"} headers:{key:"x-envoy-attempt-count" value:"1"} headers:{key:"x-ext-authz" value:"deny"} headers:{key:"x-forwarded-client-cert" value:"By=spiffe://cluster.local/ns/foo/sa/httpbin;Hash=dd14782fa2f439724d271dbed846ef843ff40d3932b615da650d028db655fc8d;Subject=\"\";URI=spiffe://cluster.local/ns/foo/sa/curl"} headers:{key:"x-forwarded-proto" value:"http"} headers:{key:"x-request-id" value:"9609691a-4e9b-9545-ac71-3889bc2dffb0"} path:"/headers" host:"httpbin.foo:8000" protocol:"HTTP/1.1"}} metadata_context:{}
   2021/01/08 03:25:06 [gRPCv3][allowed]: httpbin.foo:8000/headers, attributes: source:{address:{socket_address:{address:"10.44.0.22" port_value:52184}} principal:"spiffe://cluster.local/ns/foo/sa/curl"} destination:{address:{socket_address:{address:"10.44.3.30" port_value:80}} principal:"spiffe://cluster.local/ns/foo/sa/httpbin"} request:{time:{seconds:1610076300 nanos:925912000} http:{id:"17995949296433813435" method:"GET" headers:{key:":authority" value:"httpbin.foo:8000"} headers:{key:":method" value:"GET"} headers:{key:":path" value:"/headers"} headers:{key:"accept" value:"_/_"} headers:{key:"content-length" value:"0"} headers:{key:"user-agent" value:"curl/7.74.0-DEV"} headers:{key:"x-b3-sampled" value:"1"} headers:{key:"x-b3-spanid" value:"a66b5470e922fa80"} headers:{key:"x-b3-traceid" value:"300c2f2b90a618c8a66b5470e922fa80"} headers:{key:"x-envoy-attempt-count" value:"1"} headers:{key:"x-ext-authz" value:"allow"} headers:{key:"x-forwarded-client-cert" value:"By=spiffe://cluster.local/ns/foo/sa/httpbin;Hash=dd14782fa2f439724d271dbed846ef843ff40d3932b615da650d028db655fc8d;Subject=\"\";URI=spiffe://cluster.local/ns/foo/sa/curl"} headers:{key:"x-forwarded-proto" value:"http"} headers:{key:"x-request-id" value:"2b62daf1-00b9-97d9-91b8-ba6194ef58a4"} path:"/headers" host:"httpbin.foo:8000" protocol:"HTTP/1.1"}} metadata_context:{}
   {{< /text >}}

   また、ログから `ext-authz` フィルターとサンプルサーバー間の接続が mTLS で保護されていること（source principal に `spiffe://cluster.local/ns/foo/sa/curl` が入っている）も確認できます。

   これで、サンプル `ext-authz` サーバーに対して誰がアクセスできるかを制御する認可ポリシーも適用できます。

## クリーンアップ {#clean-up}

1. foo 名前空間を削除します：

   {{< text bash >}}
   $ kubectl delete namespace foo
   {{< /text >}}

1. メッシュ設定から拡張プロバイダー定義を削除します。

## パフォーマンス期待値 {#performance-expectations}

[パフォーマンスベンチマーク](https://github.com/istio/tools/tree/master/perf/benchmark/configs/istio/ext_authz) を参照してください。
