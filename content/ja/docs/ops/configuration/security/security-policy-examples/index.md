---
title: セキュリティポリシーの例
description: Istio セキュリティポリシーの一般的な例を紹介します。
weight: 60
owner: istio/wg-security-maintainers
test: yes
---

## 背景 {#background}

このページでは、Istio セキュリティポリシーの一般的なパターンを紹介します。
これらのパターンはデプロイ時に役立つ場合があり、ポリシー例のクイックリファレンスとしても利用できます。

ここで紹介するポリシーはあくまで例であり、実際の環境に合わせて修正が必要です。

また、[認証](/ja/docs/tasks/security/authentication/authn-policy)や[認可](/ja/docs/tasks/security/authorization)のタスクも参照し、
セキュリティポリシーの実践的な使い方を学んでください。

### ホストごとに異なる JWT 発行者が必要な場合 {#require-different-jwt-issuer-per-host}

JWT 検証は通常 Ingress Gateway で使われ、ホストごとに異なる JWT 発行者を使いたい場合があります。
[リクエスト認証](/ja/docs/tasks/security/authentication/authn-policy/#end-user-authentication)ポリシーに加え、
より細かい JWT 検証には認可ポリシーも利用できます。

JWT サブジェクトが一致する場合のみ特定ホストへのアクセスを許可したい場合、以下のポリシーを使います。
他のホストへのアクセスは常に拒否されます。

{{< text yaml >}}
apiVersion: security.istio.io/v1
kind: AuthorizationPolicy
metadata:
name: jwt-per-host
namespace: istio-system
spec:
selector:
matchLabels:
istio: ingressgateway
action: ALLOW
rules:

- from:
  - source: # JWT トークンの発行者は "@example.com" で終わる必要があります
    requestPrincipals: ["*@example.com"]
    to:
  - operation:
    hosts: ["example.com", "*.example.com"]
- from: - source: # JWT トークンの発行者は "@another.org" で終わる必要があります
  requestPrincipals: ["*@another.org"]
  to: - operation:
  hosts: [".another.org", "*.another.org"]
  {{< /text >}}

### ネームスペース分離 {#namespace-isolation}

以下の 2 つのポリシーは、ネームスペース `foo` で `STRICT` mTLS を有効にし、
同じネームスペースからのトラフィックのみを許可します。

{{< text yaml >}}
apiVersion: security.istio.io/v1
kind: PeerAuthentication
metadata:
name: default
namespace: foo
spec:
mtls:
mode: STRICT

---

apiVersion: security.istio.io/v1
kind: AuthorizationPolicy
metadata:
name: foo-isolation
namespace: foo
spec:
action: ALLOW
rules:

- from: - source:
  namespaces: ["foo"]
  {{< /text >}}

### Ingress を除外したネームスペース分離 {#namespace-isolation-with-ingress-exception}

以下の 2 つのポリシーは、ネームスペース `foo` で Strict mTLS を有効にし、
同じネームスペースおよび Ingress Gateway からのトラフィックを許可します。

{{< text yaml >}}
apiVersion: security.istio.io/v1
kind: PeerAuthentication
metadata:
name: default
namespace: foo
spec:
mtls:
mode: STRICT

---

apiVersion: security.istio.io/v1
kind: AuthorizationPolicy
metadata:
name: ns-isolation-except-ingress
namespace: foo
spec:
action: ALLOW
rules:

- from: - source:
  namespaces: ["foo"] - source:
  principals: ["cluster.local/ns/istio-system/sa/istio-ingressgateway-service-account"]
  {{< /text >}}

### 認可レイヤーで mTLS を必須にする（多層防御）{#require-mlts-in-authorization-layer}

`PeerAuthentication` で `STRICT` を設定していても、
トラフィックが実際に mTLS で保護されていることを認可レイヤーで追加チェックしたい場合、
（多層防御）

サブジェクトが空の場合、以下のポリシーはリクエストを拒否します。プレーンテキストの場合、サブジェクトは空になります。
つまり、サブジェクトが空でなければリクエストを許可します。
`"*"` は非空一致を意味し、`notPrincipals` と組み合わせると空サブジェクトにマッチします。

{{< text yaml >}}
apiVersion: security.istio.io/v1
kind: AuthorizationPolicy
metadata:
name: require-mtls
namespace: foo
spec:
action: DENY
rules:

- from: - source:
  notPrincipals: ["*"]
  {{< /text >}}

### `DENY` ポリシーで強制的な認可チェックを要求する {#require-mandatory-authorization-check-with-deny-policy}

認可チェックを必ず満たし、より緩い `ALLOW` ポリシーでバイパスされないようにしたい場合、
`DENY` ポリシーを使うことができます。`DENY` ポリシーは `ALLOW` より優先され、
`ALLOW` ポリシーより前にリクエストを拒否できます。

[リクエスト認証](/ja/docs/tasks/security/authentication/authn-policy/#end-user-authentication)ポリシーに加え、
以下のポリシーで JWT 検証を強制できます。リクエストサブジェクトが空の場合、このポリシーはリクエストを拒否します。
JWT 検証に失敗した場合、リクエストサブジェクトは空になります。つまり、リクエストサブジェクトが空でなければ許可されます。
`"*"` は非空一致を意味し、`notRequestPrincipals` と組み合わせると空リクエストサブジェクトにマッチします。

{{< text yaml >}}
apiVersion: security.istio.io/v1
kind: AuthorizationPolicy
metadata:
name: require-jwt
namespace: istio-system
spec:
selector:
matchLabels:
istio: ingressgateway
action: DENY
rules:

- from: - source:
  notRequestPrincipals: ["*"]
  {{< /text >}}

同様に、以下のポリシーは Ingress Gateway からのリクエストも許可しつつ、
強制的なネームスペース分離を実現します。ネームスペースが `foo` でなく、かつサブジェクトが
`cluster.local/ns/istio-system/sa/istio-ingressgateway-service-account` でない場合、
このポリシーはリクエストを拒否します。つまり、ネームスペースが `foo` またはサブジェクトが
`cluster.local/ns/istio-system/sa/istio-ingressgateway-service-account` の場合のみリクエストが許可されます。

{{< text yaml >}}
apiVersion: security.istio.io/v1
kind: AuthorizationPolicy
metadata:
name: ns-isolation-except-ingress
namespace: foo
spec:
action: DENY
rules:

- from: - source:
  notNamespaces: ["foo"]
  notPrincipals: ["cluster.local/ns/istio-system/sa/istio-ingressgateway-service-account"]
  {{< /text >}}
