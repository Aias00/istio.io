---
title: セキュリティの問題
description: Istio の認証、認可、セキュリティ関連の一般的な問題のトラブルシューティング方法。
force_inline_toc: true
weight: 20
keywords: [security, citadel]
aliases:
  - /zh/help/ops/security/repairing-citadel
  - /zh/help/ops/troubleshooting/repairing-citadel
  - /zh/docs/ops/troubleshooting/repairing-citadel
owner: istio/wg-security-maintainers
test: n/a
---

## エンドユーザー認証の失敗 {#end-user-authentication-fails}

Istio を使用すると、[リクエスト認証ポリシー](/ja/docs/tasks/security/authentication/authn-policy/#end-user-authentication)によってエンドユーザー認証を有効にできます。
現在、Istio の認証ポリシーがサポートするエンドユーザー認証情報は JWT です。以下はエンドユーザーの JWT 認証問題をトラブルシュートするためのガイドです。

1. `jwksUri` が設定されていない場合、JWT の発行者が URL 形式であり、
   `url + /.well-known/openid-configuration` がブラウザで開けることを確認してください。
   例えば、JWT の発行者が `https://accounts.google.com` の場合、
   `https://accounts.google.com/.well-known/openid-configuration`
   が有効な URL であり、ブラウザで開けることを確認してください。

   {{< text yaml >}}
   apiVersion: security.istio.io/v1
   kind: RequestAuthentication
   metadata:
   name: "example-3"
   spec:
   selector:
   matchLabels:
   app: httpbin
   jwtRules:

   - issuer: "testing@secure.istio.io"
     jwksUri: "{{< github_file >}}/security/tools/jwt/samples/jwks.json"
     {{< /text >}}

1. JWT トークンが HTTP リクエストヘッダーの Authorization フィールドにある場合、
   JWT トークンが有効（期限切れでない等）であることを確認してください。JWT トークンの内容は
   [jwt.io](https://jwt.io/) などのオンライン JWT デコーダーツールで確認できます。

1. `istioctl proxy-config` コマンドを使って、対象ワークロードの Envoy プロキシ設定が正しいか確認します。

   上記のポリシー例を適用した後、以下のコマンドで `listener` のインバウンドポート
   `80` の設定を確認できます。`envoy.filters.http.jwt_authn` フィルターに、ポリシーで宣言した発行者と
   JWKS 情報が含まれていることを確認してください。

   {{< text bash >}}
   $ POD=$(kubectl get pod -l app=httpbin -n foo -o jsonpath={.items..metadata.name})
   $ istioctl proxy-config listener ${POD} -n foo --port 80 --type HTTP -o json
   <redacted>
   {
   "name": "envoy.filters.http.jwt*authn",
   "typedConfig": {
   "@type": "type.googleapis.com/envoy.config.filter.http.jwt_authn.v2alpha.JwtAuthentication",
   "providers": {
   "origins-0": {
   "issuer": "testing@secure.istio.io",
   "localJwks": {
   "inlineString": "\_redacted*"
   },
   "payloadInMetadata": "testing@secure.istio.io"
   }
   },
   "rules": [
   {
   "match": {
   "prefix": "/"
   },
   "requires": {
   "requiresAny": {
   "requirements": [
   {
   "providerName": "origins-0"
   },
   {
   "allowMissing": {}
   }
   ]
   }
   }
   }
   ]
   }
   },
   <redacted>
   {{< /text >}}

## 認可が厳しすぎる、または緩すぎる {#authorization-is-too-restrictive-or-permissive}

### ポリシー YAML ファイルにタイプミスがないか確認する {#make-sure-there-are-no-typos-in-the-policy-yaml-file}

よくあるミスは、YAML ファイルで意図せず複数の要素を定義してしまうことです。例えば、以下のポリシー：

{{< text yaml >}}
apiVersion: security.istio.io/v1
kind: AuthorizationPolicy
metadata:
name: example
namespace: foo
spec:
action: ALLOW
rules:

- to:
  - operation:
    paths:
    - /foo
- from: - source:
  namespaces: - foo
  {{< /text >}}

このポリシーで許可されるリクエストは、パスが `/foo` **かつ** ソースネームスペースが `foo` であるものを想定しているかもしれません。
しかし、実際にはパスが `/foo` **または** ソースネームスペースが `foo` であれば許可されるため、意図よりも緩い設定になります。

YAML の構文上、`from:` の前の `-` は新しいリスト要素を意味します。
これにより、ポリシー内に 2 つのルールが作成され、認証ポリシーでは複数ルール間は `OR` で評価されます。

この問題を解決するには、余分な `-` を削除し、1 つのルール内でパスが `/foo` **かつ** ソースネームスペースが `foo` のリクエストのみを許可するようにします。

### TCP ポートで HTTP 専用フィールドを使っていないか確認する {#make-sure-you-are-not-using-http-only-fields-on-tcp-ports}

認可ポリシーで HTTP 専用フィールド（`host`、`path`、`headers`、JWT など）を TCP 接続に対して指定すると、
これらのフィールドは TCP では存在しないため、ポリシーがより厳しくなります。

`ALLOW` ポリシーの場合、これらのフィールドはマッチしませんが、`DENY` や `CUSTOM` ポリシーの場合、
常にマッチしたとみなされます。その結果、意図せず接続が拒否されることがあります。

Kubernetes サービス定義を確認し、[正しいプロトコル名がポート名に含まれている](/ja/docs/ops/configuration/traffic-management/protocol-selection/#manual-protocol-selection)ことを確認してください。
HTTP 専用フィールドを使う場合は、ポート名に `http-` プレフィックスを付けてください。

### ポリシーが正しいターゲットに適用されているか確認する {#make-sure-the-policy-is-applied-to-the-correct-target}

ワークロードのセレクターやネームスペースを確認し、ポリシーが正しいターゲットに適用されているか確認してください。
`istioctl x authz check POD-NAME.POD-NAMESPACE` コマンドで認可ポリシーを確認できます。

### ポリシーで指定したアクションに注意する {#pay-attention-to-the-action-specified-in-the-policy}

- 明示しない場合、ポリシーのデフォルトアクションは `ALLOW` です。

- 1 つのワークロードに複数のアクション（`CUSTOM`、`ALLOW`、`DENY`）が設定されている場合、
  すべてのアクションが満たされる必要があります。つまり、いずれかのアクションがリクエストを拒否した場合、そのリクエストは拒否され、
  すべてのアクションが許可した場合のみリクエストが許可されます。

- どのような場合でも、`AUDIT` アクションはアクセス制御を実施せず、リクエストを拒否しません。

[認可の暗黙的有効化](/ja/docs/concepts/security/#implicit-enablement)を参照し、評価順序の詳細を確認してください。

## Istiod がポリシーを受け入れているか確認する {#ensure-istiod-accepts-the-policies}

Istiod は認可ポリシーを変換し、Sidecar に配布します。Istiod が正しく動作しているか確認するには、以下の手順を実行します：

1. 次のコマンドで Istiod のデバッグログを有効にします：

   {{< text bash >}}
   $ istioctl admin log --level authorization:debug
   {{< /text >}}

1. 次のコマンドで Istio のログを取得します：

   {{< tip >}}
   認可ポリシーを一度削除して再作成すると、デバッグログが正しく出力される場合があります。
   {{< /tip >}}

   {{< text bash >}}
   $ kubectl logs $(kubectl -n istio-system get pods -l app=istiod -o jsonpath='{.items[0].metadata.name}') -c discovery -n istio-system
   {{< /text >}}

1. 出力を確認し、エラーがないか検証します。例えば、次のような出力が見られる場合があります：

   {{< text plain >}}
   2021-04-23T20:53:29.507314Z info ads Push debounce stable[31] 1: 100.981865ms since last change, 100.981653ms since last push, full=true
   2021-04-23T20:53:29.507641Z info ads XDS: Pushing:2021-04-23T20:53:29Z/23 Services:15 ConnectedEndpoints:2 Version:2021-04-23T20:53:29Z/23
   2021-04-23T20:53:29.507911Z debug authorization Processed authorization policy for httpbin-74fb669cc6-lpscm.foo with details:
   _ found 0 CUSTOM actions
   2021-04-23T20:53:29.508077Z debug authorization Processed authorization policy for curl-557747455f-6dxbl.foo with details:
   _ found 0 CUSTOM actions
   2021-04-23T20:53:29.508128Z debug authorization Processed authorization policy for httpbin-74fb669cc6-lpscm.foo with details:
   _ found 1 DENY actions, 0 ALLOW actions, 0 AUDIT actions
   _ generated config from rule ns[foo]-policy[deny-path-headers]-rule[0] on HTTP filter chain successfully
   _ built 1 HTTP filters for DENY action
   _ added 1 HTTP filters to filter chain 0
   _ added 1 HTTP filters to filter chain 1
   2021-04-23T20:53:29.508158Z debug authorization Processed authorization policy for curl-557747455f-6dxbl.foo with details:
   _ found 0 DENY actions, 0 ALLOW actions, 0 AUDIT actions
   2021-04-23T20:53:29.509097Z debug authorization Processed authorization policy for curl-557747455f-6dxbl.foo with details:
   _ found 0 CUSTOM actions
   2021-04-23T20:53:29.509167Z debug authorization Processed authorization policy for curl-557747455f-6dxbl.foo with details:
   _ found 0 DENY actions, 0 ALLOW actions, 0 AUDIT actions
   2021-04-23T20:53:29.509501Z debug authorization Processed authorization policy for httpbin-74fb669cc6-lpscm.foo with details:
   _ found 0 CUSTOM actions
   2021-04-23T20:53:29.509652Z debug authorization Processed authorization policy for httpbin-74fb669cc6-lpscm.foo with details:
   _ found 1 DENY actions, 0 ALLOW actions, 0 AUDIT actions
   _ generated config from rule ns[foo]-policy[deny-path-headers]-rule[0] on HTTP filter chain successfully
   _ built 1 HTTP filters for DENY action
   _ added 1 HTTP filters to filter chain 0
   _ added 1 HTTP filters to filter chain 1
   _ generated config from rule ns[foo]-policy[deny-path-headers]-rule[0] on TCP filter chain successfully
   _ built 1 TCP filters for DENY action
   _ added 1 TCP filters to filter chain 2
   _ added 1 TCP filters to filter chain 3 \* added 1 TCP filters to filter chain 4
   2021-04-23T20:53:29.510903Z info ads LDS: PUSH for node:curl-557747455f-6dxbl.foo resources:18 size:85.0kB
   2021-04-23T20:53:29.511487Z info ads LDS: PUSH for node:httpbin-74fb669cc6-lpscm.foo resources:18 size:86.4kB
   {{< /text >}}

   上記の出力は、Istiod が以下の設定を生成したことを示しています：

   - ワークロード `httpbin-74fb669cc6-lpscm.foo` 用で、ポリシー
     `ns[foo]-policy[deny-path-headers]-rule[0]` を持つ HTTP フィルター設定。

   - ワークロード `httpbin-74fb669cc6-lpscm.foo` 用で、ポリシー
     `ns[foo]-policy[deny-path-headers]-rule[0]` を持つ TCP フィルター設定。

## Istiod が正しくプロキシにポリシーを配布しているか確認する {#ensure-istiod-distributes-policies-to-proxies-correctly}

Pilot はプロキシに認可ポリシーを配布します。Pilot が正しく動作しているか確認するには、以下の手順を実行します：

{{< tip >}}
以下のコマンドは [Bookinfo](/ja/docs/examples/bookinfo/) をデプロイしていることを前提としています。
`httpbin` を使っていない場合は、`"-l app=httpbin"` を実際の値に置き換えてください。
{{< /tip >}}

1. 次のコマンドで `httpbin` ワークロードのプロキシ設定を取得します：

   {{< text bash >}}
   $ kubectl exec $(kubectl get pods -l app=httpbin -o jsonpath='{.items[0].metadata.name}') -c istio-proxy -- pilot-agent request GET config_dump
   {{< /text >}}

1. ログ内容を確認します：

   - ログに `envoy.filters.http.rbac` フィルターが含まれており、すべてのリクエストに対して認可ポリシーが強制されていること。
   - 認可ポリシーを更新すると、Istio がフィルターを更新すること。

1. 以下の出力例は、`httpbin` のプロキシで `envoy.filters.http.rbac` フィルターが有効であり、
   `/headers` パスへのアクセスをすべて拒否するルールが設定されていることを示しています。

   {{< text plain >}}
   {
   "name": "envoy.filters.http.rbac",
   "typed*config": {
   "@type": "type.googleapis.com/envoy.extensions.filters.http.rbac.v3.RBAC",
   "rules": {
   "action": "DENY",
   "policies": {
   "ns[foo]-policy[deny-path-headers]-rule[0]": {
   "permissions": [
   {
   "and_rules": {
   "rules": [
   {
   "or_rules": {
   "rules": [
   {
   "url_path": {
   "path": {
   "exact": "/headers"
   }
   }
   }
   ]
   }
   }
   ]
   }
   }
   ],
   "principals": [
   {
   "and_ids": {
   "ids": [
   {
   "any": true
   }
   ]
   }
   }
   ]
   }
   }
   },
   "shadow_rules_stat_prefix": "istio_dry_run_allow*"
   }
   },
   {{< /text >}}

## プロキシでポリシーが正しく適用されているか確認する {#ensure-proxies-enforce-policies-correctly}

プロキシは認可ポリシーの最終的な実施者です。プロキシが正しく動作しているか確認するには、以下の手順を実行します：

{{< tip >}}
以下のコマンドは `httpbin` をデプロイしていることを前提としています。
`httpbin` を使っていない場合は、`"-l app=httpbin"` を実際の Pod 名に置き換えてください。
{{< /tip >}}

1. 次のコマンドでプロキシの認可デバッグログを有効にします：

   {{< text bash >}}
   $ istioctl proxy-config log deploy/httpbin --level "rbac:debug"
   {{< /text >}}

1. 以下の出力が表示されることを確認します：

   {{< text plain >}}
   active loggers:
   ... ...
   rbac: debug
   ... ...
   {{< /text >}}

1. `httpbin` ワークロードにリクエストを送信し、ログを生成します。

1. 次のコマンドでプロキシのログを表示します：

   {{< text bash >}}
   $ kubectl logs $(kubectl get pods -l app=httpbin -o jsonpath='{.items[0].metadata.name}') -c istio-proxy
   {{< /text >}}

1. 出力を確認し、以下を検証します：

   - リクエストが許可または拒否された場合、それぞれ `enforced allowed` または `enforced denied` のログが出力されること。
   - 認可ポリシーがリクエストからデータを取得していること。

1. 以下は `/httpbin` パスへのリクエスト時の出力例です：

   {{< text plain >}}
   ...
   2021-04-23T20:43:18.552857Z debug envoy rbac checking request: requestedServerName: outbound*.8000*.\_.httpbin.foo.svc.cluster.local, sourceIP: 10.44.3.13:46180, directRemoteIP: 10.44.3.13:46180, remoteIP: 10.44.3.13:46180,localAddress: 10.44.1.18:80, ssl: uriSanPeerCertificate: spiffe://cluster.local/ns/foo/sa/curl, dnsSanPeerCertificate: , subjectPeerCertificate: , headers: ':authority', 'httpbin:8000'
   ':path', '/headers'
   ':method', 'GET'
   ':scheme', 'http'
   'user-agent', 'curl/7.76.1-DEV'
   'accept', '_/_'
   'x-forwarded-proto', 'http'
   'x-request-id', '672c9166-738c-4865-b541-128259cc65e5'
   'x-envoy-attempt-count', '1'
   'x-b3-traceid', '8a124905edf4291a21df326729b264e9'
   'x-b3-spanid', '21df326729b264e9'
   'x-b3-sampled', '0'
   'x-forwarded-client-cert', 'By=spiffe://cluster.local/ns/foo/sa/httpbin;Hash=d64cd6750a3af8685defbbe4dd8c467ebe80f6be4bfe9ca718e81cd94129fc1d;Subject="";URI=spiffe://cluster.local/ns/foo/sa/curl'
   , dynamicMetadata: filter_metadata {
   key: "istio_authn"
   value {
   fields {
   key: "request.auth.principal"
   value {
   string_value: "cluster.local/ns/foo/sa/curl"
   }
   }
   fields {
   key: "source.namespace"
   value {
   string_value: "foo"
   }
   }
   fields {
   key: "source.principal"
   value {
   string_value: "cluster.local/ns/foo/sa/curl"
   }
   }
   fields {
   key: "source.user"
   value {
   string_value: "cluster.local/ns/foo/sa/curl"
   }
   }
   }
   }

   2021-04-23T20:43:18.552910Z debug envoy rbac enforced denied, matched policy ns[foo]-policy[deny-path-headers]-rule[0]
   ...
   {{< /text >}}

   ログ `enforced denied, matched policy ns[foo]-policy[deny-path-headers]-rule[0]`
   は、リクエストがポリシー `ns[foo]-policy[deny-path-headers]-rule[0]` によって拒否されたことを示します。

1. [ドライランモード](/ja/docs/tasks/security/authorization/authz-dry-run)での認可ポリシーの出力例：

   {{< text plain >}}
   ...
   2021-04-23T20:59:11.838468Z debug envoy rbac checking request: requestedServerName: outbound*.8000*.\_.httpbin.foo.svc.cluster.local, sourceIP: 10.44.3.13:49826, directRemoteIP: 10.44.3.13:49826, remoteIP: 10.44.3.13:49826,localAddress: 10.44.1.18:80, ssl: uriSanPeerCertificate: spiffe://cluster.local/ns/foo/sa/curl, dnsSanPeerCertificate: , subjectPeerCertificate: , headers: ':authority', 'httpbin:8000'
   ':path', '/headers'
   ':method', 'GET'
   ':scheme', 'http'
   'user-agent', 'curl/7.76.1-DEV'
   'accept', '_/_'
   'x-forwarded-proto', 'http'
   'x-request-id', 'e7b2fdb0-d2ea-4782-987c-7845939e6313'
   'x-envoy-attempt-count', '1'
   'x-b3-traceid', '696607fc4382b50017c1f7017054c751'
   'x-b3-spanid', '17c1f7017054c751'
   'x-b3-sampled', '0'
   'x-forwarded-client-cert', 'By=spiffe://cluster.local/ns/foo/sa/httpbin;Hash=d64cd6750a3af8685defbbe4dd8c467ebe80f6be4bfe9ca718e81cd94129fc1d;Subject="";URI=spiffe://cluster.local/ns/foo/sa/curl'
   , dynamicMetadata: filter_metadata {
   key: "istio_authn"
   value {
   fields {
   key: "request.auth.principal"
   value {
   string_value: "cluster.local/ns/foo/sa/curl"
   }
   }
   fields {
   key: "source.namespace"
   value {
   string_value: "foo"
   }
   }
   fields {
   key: "source.principal"
   value {
   string_value: "cluster.local/ns/foo/sa/curl"
   }
   }
   fields {
   key: "source.user"
   value {
   string_value: "cluster.local/ns/foo/sa/curl"
   }
   }
   }
   }

   2021-04-23T20:59:11.838529Z debug envoy rbac shadow denied, matched policy ns[foo]-policy[deny-path-headers]-rule[0]
   2021-04-23T20:59:11.838538Z debug envoy rbac no engine, allowed by default
   ...
   {{< /text >}}

   ログ `shadow denied, matched policy ns[foo]-policy[deny-path-headers]-rule[0]`
   は、リクエストが**ドライラン**ポリシー `ns[foo]-policy[deny-path-headers]-rule[0]` によって拒否されることを示します。

   ログ `no engine, allowed by default` は、ドライランポリシーがワークロード上で唯一のポリシーであるため、
   実際にはリクエストが許可されたことを示します。

## 鍵と証明書のエラー {#keys-and-certificates-errors}

Istio で使用されている鍵や証明書に問題があると疑われる場合、任意の Pod の内容を確認できます。

{{< text bash >}}
$ istioctl proxy-config secret curl-8f795f47d-4s4t7
RESOURCE NAME TYPE STATUS VALID CERT SERIAL NUMBER NOT AFTER NOT BEFORE
default Cert Chain ACTIVE true 138092480869518152837211547060273851586 2020-11-11T16:39:48Z 2020-11-10T16:39:48Z
ROOTCA CA ACTIVE true 288553090258624301170355571152070165215 2030-11-08T16:34:52Z 2020-11-10T16:34:52Z
{{< /text >}}

`-o json` オプションを使うと、証明書の全内容を `openssl` で解析できます：

{{< text bash >}}
$ istioctl proxy-config secret curl-8f795f47d-4s4t7 -o json | jq '[.dynamicActiveSecrets[] | select(.name == "default")][0].secret.tlsCertificate.certificateChain.inlineBytes' -r | base64 -d | openssl x509 -noout -text
Certificate:
Data:
Version: 3 (0x2)
Serial Number:
99:59:6b:a2:5a:f4:20:f4:03:d7:f0:bc:59:f5:d8:40
Signature Algorithm: sha256WithRSAEncryption
Issuer: O = k8s.cluster.local
Validity
Not Before: Jun 4 20:38:20 2018 GMT
Not After : Sep 2 20:38:20 2018 GMT
...
X509v3 extensions:
X509v3 Key Usage: critical
Digital Signature, Key Encipherment
X509v3 Extended Key Usage:
TLS Web Server Authentication, TLS Web Client Authentication
X509v3 Basic Constraints: critical
CA:FALSE
X509v3 Subject Alternative Name:
URI:spiffe://cluster.local/ns/my-ns/sa/my-sa
...
{{< /text >}}

表示された証明書に有効な情報が含まれていることを確認してください。特に、`Subject Alternative Name` フィールドが
`URI:spiffe://cluster.local/ns/my-ns/sa/my-sa` であることを確認してください。

## 双方向 TLS のエラー {#mutual-TLS-errors}

双方向 TLS に問題がある場合、まず istiod の正常性を確認し、
次に [鍵と証明書が正しく Sidecar に配布されているか](#keys-and-certificates-errors)を確認してください。

上記の確認で問題がなければ、次に[認証ポリシー](/ja/docs/tasks/security/authentication/authn-policy/)が作成されているか、
また、ターゲットのルールが正しく適用されているかを検証してください。

クライアント Sidecar が正しく双方向 TLS またはプレーンテキストトラフィックを送信していないと疑われる場合は、
[Grafana ワークロードダッシュボード](/ja/docs/ops/integrations/grafana/)を確認してください。
mTLS の有無にかかわらず、送信リクエストには注釈が付与されます。確認後、クライアント Sidecar に問題があると思われる場合は、
GitHub で Issue を作成してください。
