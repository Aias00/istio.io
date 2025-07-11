---
title: セキュリティのベストプラクティス
description: Istio でアプリケーションを保護するためのベストプラクティス。
force_inline_toc: true
weight: 30
owner: istio/wg-security-maintainers
test: n/a
---

Istio のセキュリティ機能は、強力な ID、堅牢なポリシー、透過的な TLS 暗号化、認証、認可、監査（AAA）ツールを提供し、サービスとデータを保護します。しかし、これらのセキュリティ機能を最大限に活用するには、ベストプラクティスに従う必要があります。まずは[セキュリティの概要](/ja/docs/concepts/security/)を確認してから、以下をお読みください。

## 双方向 TLS {#mutual-tls}

Istio は可能な限り[自動的に](/ja/docs/ops/configuration/traffic-management/tls-configuration/#auto-mtls)トラフィックを[双方向 TLS](/ja/docs/concepts/security/#mutual-tls-authentication)で暗号化します。ただし、デフォルトではプロキシは[パーミッシブ（PERMISSIVE）モード](/ja/docs/concepts/security/#permissive-mode)で動作し、双方向 TLS 認証トラフィックとプレーンテキストトラフィックの両方を許可します。

このモードは段階的な導入や Istio プロキシを持たないクライアントからのトラフィックを許可するために必要ですが、セキュリティを弱めることにもなります。そのため、トラフィックを双方向 TLS 認証に強制するには、できるだけ早く[ストリクト（STRICT）モード](/ja/docs/tasks/security/authentication/mtls-migration/)へ移行することを推奨します。

双方向 TLS だけでは安全なトラフィックを保証できません。認証のみで認可は行われないため、有効な証明書を持つ者は誰でもアクセスできます。

本当に安全なトラフィックを実現するには、[認可ポリシー](/ja/docs/tasks/security/authorization/)も併せて設定してください。これにより、細かなポリシーでトラフィックの許可・拒否を制御できます。たとえば、`app` 名前空間からのリクエストだけが `hello-world` ワークロードにアクセスできるように設定できます。

## 認可ポリシー {#authorization-policies}

Istio の[認可](/ja/docs/concepts/security/#authorization)は、Istio セキュリティの中核です。適切な認可ポリシーを設定することで、クラスタを最大限に保護できます。Istio はユーザーごとに最適な認可ポリシーを決めることはできないため、以下の内容をよく理解してください。

### より安全な認可ポリシーパターン {#safer-authorization-policy-patterns}

#### default-deny 認可ポリシーパターンの利用 {#use-default-deny-patterns}

Istio のポリシーはデフォルト拒否（default-deny）に設定することを推奨します。default-deny 認可ポリシーは、デフォルトですべてのリクエストを拒否し、許可する条件を明示的に定義します。条件を定義し忘れた場合、該当トラフィックは拒否されます（意図しない許可よりも安全です）。

たとえば、[HTTP トラフィック認可タスク](/ja/docs/tasks/security/authorization/authz-http/)の `allow-nothing` 認可ポリシーは、デフォルトですべてのトラフィックを拒否します。その上で、必要なトラフィックのみを許可する認可ポリシーを追加できます。

#### waypoint の default-deny パターン {#default-deny-pattern-with-waypoints}

Istio の新しい Ambient データプレーンモードでは、分割型データプレーンアーキテクチャが導入されました。このアーキテクチャでは、waypoint プロキシは Kubernetes Gateway API 設定を使い、`parentRef` と `targetRef` で Gateway に明示的にバインドします。waypoint は Kubernetes Gateway API の原則に厳密に従うため、waypoint へのポリシー適用時の default-deny パターンの有効化方法がやや異なります。Istio 1.25 以降、`AuthorizationPolicy` リソースを `istio-waypoint` の `GatewayClass` にバインドできます。`GatewayClass` はクラスタ全体のリソースなので、バインドするポリシーは通常 `istio-system` などのルート名前空間に配置する必要があります。

{{< tip >}}
waypoint で default-deny パターンを使う場合は、「クラシック」な default-deny ポリシーに加え、`istio-waypoint` `GatewayClass` にバインドしたポリシーも使いましょう。ztunnel はメッシュ内ワークロードに対して「クラシック」default-deny ポリシーを強制し、有効な値を提供し続けます。
{{< /tip >}}

#### `ALLOW-with-positive-matching` と `DENY-with-negative-match` パターンの利用 {#use-allow-with-positive-matching-and-deny-with-negative-match-patterns}

可能な限り `ALLOW-with-positive-matching` または `DENY-with-negative-matching` 認可ポリシーパターンを使いましょう。これらのパターンは、ポリシーがマッチしない場合でも最悪 403 拒否となり、認可ポリシーのバイパスを防げるため安全です。

`ALLOW-with-positive-matching` パターンは、**positive** マッチフィールド（`paths`、`values` など）でのみ `ALLOW` アクションを使い、**negative** マッチフィールド（`notPaths`、`notValues` など）は使いません。

`DENY-with-negative-matching` パターンは、**negative** マッチフィールド（`notPaths`、`notValues` など）でのみ `DENY` アクションを使い、**positive** マッチフィールド（`paths`、`values` など）は使いません。

たとえば、次の認可ポリシーは `ALLOW-with-positive-matching` パターンで、パス `/public` へのリクエストのみを許可します：

{{< text yaml >}}
apiVersion: security.istio.io/v1
kind: AuthorizationPolicy
metadata:
name: foo
spec:
action: ALLOW
rules:

- to: - operation:
  paths: ["/public"]
  {{< /text >}}

このポリシーは許可するパス（`/public`）を明示的に列挙しています。つまり、リクエストパスが `/public` と一致した場合のみ許可され、それ以外はデフォルトで拒否されます。これにより、未知の正規化動作によるポリシーバイパスのリスクがなくなります。

同じ結果を得る `DENY-with-negative-matching` パターンの例：

{{< text yaml >}}
apiVersion: security.istio.io/v1
kind: AuthorizationPolicy
metadata:
name: foo
spec:
action: DENY
rules:

- to: - operation:
  notPaths: ["/public"]
  {{< /text >}}

### 認可ポリシーにおけるパス正規化の理解 {#understand-path-normalization-in-authorization-policy}

認可ポリシーの実行ポイントは Envoy プロキシであり、通常のリソースアクセス点（バックエンドアプリ）ではありません。Envoy プロキシとバックエンドアプリでリクエストの解釈が異なる場合、ポリシーの不一致が発生します。

ポリシーの不一致は、意図しない拒否やポリシーバイパスにつながります。後者はセキュリティインシデントであり、即時修正が必要です。そのため、認可ポリシーでパス正規化が重要です。

たとえば、パス `/data/secret` を拒否する認可ポリシーを考えます。パス `/data//secret` のリクエストは拒否されません（パスに余分なスラッシュがあるため）。

リクエストがバックエンドアプリに到達すると、アプリは `/data//secret` を `/data/secret` に正規化し、同じレスポンスを返します。

この例では、ポリシー実行点（Envoy）とリソースアクセス点（アプリ）でパスの解釈が異なり、不一致が発生し、認可ポリシーがバイパスされます。

この問題が複雑なのは、

- 明確な正規化標準がない
- バックエンドやフレームワークごとに独自の正規化がある
- アプリが独自の正規化を行う場合もある

ためです。

Istio の認可ポリシーは、さまざまな基本的な正規化オプションをサポートしています：

- [パス正規化オプションの設定ガイド](/ja/docs/ops/best-practices/security/#guideline-on-configuring-the-path-normalization-option)で利用可能な正規化オプションを確認してください。
- [システムごとのパス正規化カスタマイズ](/ja/docs/ops/best-practices/security/#customize-your-system-on-path-normalization)で各オプションの詳細を確認してください。
- サポートされていない正規化が必要な場合は、[未サポート正規化の緩和策](/ja/docs/ops/best-practices/security/#mitigation-for-unsupported-normalization)を参照してください。

### パス正規化オプション設定のガイドライン {#guideline-on-configuring-the-path-normalization-option}

#### ケース 1：正規化が不要な場合 {#case-1-you-do-not-need-normalization-at-all}

まず、正規化が必要かどうかを判断してください。

認可ポリシーを使わない、または `path` フィールドを使わない場合は正規化不要です。

すべての認可ポリシーが[より安全な認可パターン](/ja/docs/ops/best-practices/security/#safer-authorization-policy-patterns)に従っていれば、最悪でも意図しない拒否となり、バイパスは発生しません。

#### ケース 2：正規化が必要だが、どのオプションを使うべきか分からない場合 {#case-2-you-need-normalization-but-not-sure-which-normalization-option-to-use}

正規化が必要だが、どのオプションを使うべきか分からない場合は、最も厳格な正規化オプションを選ぶのが安全です。

複雑な多層システムでは、リクエストの外で何が正規化されているか把握するのは困難です。

要件を満たし、意味が明確であれば、より緩やかな正規化オプションも使えます。

いずれの場合も、要件に合わせて正・負のテストを作成し、正規化が期待通りに動作するか検証してください。

[システムごとのパス正規化カスタマイズ](/ja/docs/ops/best-practices/security/#customize-your-system-onpath-normalization)も参照してください。

#### ケース 3：未サポートの正規化オプションが必要な場合 {#case-3-you-need-an-unsupported-normalization-option}

Istio でまだサポートされていない正規化オプションが必要な場合は、[未サポート正規化の緩和策](/ja/docs/ops/best-practices/security/#mitigation-for-unsupported-normalization)を参照し、カスタム正規化や Istio コミュニティへの機能リクエストを検討してください。

### パス正規化のカスタマイズ {#customize-your-system-on-path-normalization}

Istio の認可ポリシーは HTTP リクエストの URL パスに基づいて動作します。
[パス正規化（URI 正規化）](https://en.wikipedia.org/wiki/URI_normalization)は、受信リクエストのパスを修正・標準化し、標準的な方法で処理できるようにします。構文上は異なるパスでも、正規化後は一致する場合があります。

認可ポリシーやリクエストルーティングの評価前に、Istio は以下のパス正規化方式をサポートします：

| オプション                 | 説明                                                                                                                                                                                                                                                                                                                                                                                                                                          | 例                                                                 |
| -------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------ |
| `NONE`                     | 正規化なし。Envoy プロキシが受け取ったものをそのままバックエンドに転送。                                                                                                                                                                                                                                                                                                                                                                      | `../%2Fa../b` は認可ポリシー評価・バックエンド転送ともにそのまま。 |
| `BASE`                     | Istio の**デフォルト**インストールオプション。Envoy プロキシで[パス正規化](https://www.envoyproxy.io/docs/envoy/latest/api-v3/extensions/filters/network/http_connection_manager/v3/http_connection_manager.proto#envoy-v3-api-field-extensions-filters-network-http-connection-manager-v3-httpconnectionmanager-normalize-path)を適用。これは [RFC 3986](https://tools.ietf.org/html/rfc3986) に準拠し、バックスラッシュをスラッシュに変換。 | `/a/../b` → `/b`、`\da` → `/da`                                    |
| `MERGE_SLASHES`            | **BASE** 正規化後にスラッシュをマージ。                                                                                                                                                                                                                                                                                                                                                                                                       | `/a//b` → `/a/b`                                                   |
| `DECODE_AND_MERGE_SLASHES` | すべてのトラフィックをデフォルト許可する場合に最も厳格な設定。ルーティングや認可ポリシーを厳密にテストしたい場合に推奨。`MERGE_SLASHES` 前に[パーセントエンコード](https://tools.ietf.org/html/rfc3986#section-2.1)されたスラッシュやバックスラッシュ（`%2F`、`%2f`、`%5C`、`%5c`）をデコード。                                                                                                                                               | `/a%2fb` → `/a/b`                                                  |

{{< tip >}}
この設定は[メッシュ設定](/ja/docs/reference/config/istio.mesh.v1alpha1/)の
[`pathNormalization`](/ja/docs/reference/config/istio.mesh.v1alpha1/#MeshConfig-ProxyPathNormalization)
フィールドで宣言します。
{{< /tip >}}

正規化アルゴリズムは以下の順で実行されます：

1. `%2F`、`%2f`、`%5C`、`%5c` のパーセントデコード
1. [RFC 3986](https://tools.ietf.org/html/rfc3986) および Envoy [`normalize_path`](https://www.envoyproxy.io/docs/envoy/latest/api-v3/extensions/filters/network/http_connection_manager/v3/http_connection_manager.proto#envoy-v3-api-field-extensions-filters-network-http-connection-manager-v3-httpconnectionmanager-normalize-path) オプションによる正規化
1. スラッシュのマージ

{{< warning >}}
これらの正規化オプションは HTTP 標準および業界推奨設定ですが、アプリケーションが独自の URL を使う場合もあります。否定的なポリシーを使う場合は、アプリの動作を十分理解してください。
{{< /warning >}}

サポートされている正規化一覧は[認可ポリシー仕様](/ja/docs/reference/config/security/normalization/)を参照してください。

### 設定例 {#examples-of-configuration}

Envoy でのリクエストパス正規化がバックエンドの期待と一致していることを確認することが重要です。以下は参考例です。
正規化済みまたは `NONE` 選択時の元の URL パスは：

1. 認可ポリシーのチェックに使われる
1. バックエンドアプリに転送される

| アプリの動作...                                                                                                              | 選択肢...                                           |
| ---------------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------- |
| プロキシに正規化を任せる                                                                                                     | `BASE`、`MERGE_SLASHES`、`DECODE_AND_MERGE_SLASHES` |
| [RFC 3986](https://tools.ietf.org/html/rfc3986) で正規化しスラッシュはマージしない                                           | `BASE`                                              |
| [RFC 3986](https://tools.ietf.org/html/rfc3986) で正規化しスラッシュはマージ、パーセントエンコードスラッシュはデコードしない | `MERGE_SLASHES`                                     |
| [RFC 3986](https://tools.ietf.org/html/rfc3986) で正規化しスラッシュはマージ、パーセントエンコードスラッシュもデコード       | `DECODE_AND_MERGE_SLASHES`                          |
| [RFC 3986](https://tools.ietf.org/html/rfc3986) 非互換のパス処理                                                             | `NONE`                                              |

### 設定方法 {#how-to-configure}

`istioctl` コマンドで[メッシュ設定](/ja/docs/reference/config/istio.mesh.v1alpha1/)を更新できます：

{{< text bash >}}
$ istioctl upgrade --set meshConfig.pathNormalization.normalization=DECODE_AND_MERGE_SLASHES
{{< /text >}}

または Operator でファイルを上書き：

{{< text bash >}}
$ cat <<EOF > iop.yaml
apiVersion: install.istio.io/v1alpha1
kind: IstioOperator
spec:
meshConfig:
pathNormalization:
normalization: DECODE_AND_MERGE_SLASHES
EOF
$ istioctl install -f iop.yaml
{{< /text >}}

または、メッシュ設定を直接編集する場合は、
[`pathNormalization`](/ja/docs/reference/config/istio.mesh.v1alpha1/#MeshConfig-ProxyPathNormalization)
を[メッシュ設定](/ja/docs/reference/config/istio.mesh.v1alpha1/)に追加し、`istio-system` 名前空間の `istio-<REVISION_ID>` configmap を編集します。たとえば `DECODE_AND_MERGE_SLASHES` を使う場合：

{{< text yaml >}}
apiVersion: v1
data:
mesh: |-
...
pathNormalization:
normalization: DECODE_AND_MERGE_SLASHES
...
{{< /text >}}

### 未サポート正規化の緩和策 {#mitigation-for-unsupported-normalization}

Istio でサポートされていない正規化が必要な場合の緩和策を紹介します。これらは Istio の範囲外のものも含むため、十分理解し注意して使ってください。

#### カスタム正規化ロジック {#custom-normalization-logic}

WASM または Lua フィルタでカスタム正規化ロジックを適用できます。WASM フィルタは Istio 公式サポートなので推奨です。Lua フィルタは PoC には使えますが、本番利用は推奨されません。

#### 大文字小文字の正規化 {#case-normalization}

一部環境では、認可ポリシーのパスで大文字小文字を区別しない必要があります。
たとえば `https://myurl/get` と `https://myurl/GeT` を同一視したい場合です。

この場合、以下のような `EnvoyFilter` を使えます。この設定はポリシー比較とアプリ転送の両方でパスを変更します。

{{< text syntax=yaml snip_id=ingress_case_insensitive_envoy_filter >}}
apiVersion: networking.istio.io/v1alpha3
kind: EnvoyFilter
metadata:
name: ingress-case-insensitive
namespace: istio-system
spec:
configPatches:

- applyTo: HTTP_FILTER
  match:
  context: GATEWAY
  listener:
  filterChain:
  filter:
  name: "envoy.filters.network.http_connection_manager"
  patch:
  operation: INSERT_FIRST
  value:
  name: envoy.lua
  typed_config:
  "@type": "type.googleapis.com/envoy.extensions.filters.http.lua.v3.Lua"
  inlineCode: |
  function envoy_on_request(request_handle)
  local path = request_handle:headers():get(":path")
  request_handle:headers():replace(":path", string.lower(path))
  end
  {{< /text >}}

#### ホストマッチポリシーの記述 {#writing-host-match-policies}

Istio はホスト名自体と、すべてのマッチするポートに対してホスト名を生成します。たとえば、仮想サービスやゲートウェイは `example.com` と `example.com:*` の両方にマッチする設定を生成します。ただし、完全一致の認可ポリシーは `hosts` や `notHosts` フィールドに指定した文字列とだけ一致します。

[認可ポリシールール](/ja/docs/reference/config/security/authorization-policy/#Rule)でマッチさせるホストは、完全一致ではなくプレフィックスマッチを使うべきです。たとえば、`example.com` 用の Envoy 設定にマッチさせるには、`hosts: ["example.com", "example.com:*"]` のようにします：

{{< text yaml >}}
apiVersion: security.istio.io/v1
kind: AuthorizationPolicy
metadata:
name: ingress-host
namespace: istio-system
spec:
selector:
matchLabels:
app: istio-ingressgateway
action: DENY
rules:

- to: - operation:
  hosts: ["example.com", "example.com:*"]
  {{< /text >}}

また、`host` や `notHosts` フィールドは通常、メッシュ外部からのトラフィックを受けるゲートウェイでのみ使い、メッシュ内トラフィックの Sidecar では使うべきではありません。なぜなら、サーバー側 Sidecar（認可ポリシーの実行点）はリクエストをアプリに転送する際に `Host` フィールドを使わないからです。クライアントは明示的な IP アドレスや任意の `Host` ヘッダでアプリにアクセスできるため、Sidecar での `host` や `notHost` は意味を持ちません。

Sidecar で `Host` ヘッダに基づくアクセス制御が必要な場合は、[default-deny 認可ポリシーパターン](/ja/docs/ops/best-practices/security/#use-default-deny-patterns)を使い、任意の `Host` ヘッダを使うクライアントからのリクエストは拒否されるようにしてください。

#### 専用の Web アプリケーションファイアウォール（WAF） {#specialized-web-application-firewall}

多くの Web アプリケーションファイアウォール（WAF）製品は追加の正規化オプションを提供します。これらは Istio Ingress Gateway の前段に配置し、メッシュに入るリクエストを正規化できます。その後、認可ポリシーは正規化済みリクエストに対して適用されます。詳細は各 WAF 製品のドキュメントを参照してください。

#### Istio への機能リクエスト {#feature-request-to-istio}

Istio で特定の正規化を公式サポートしてほしい場合は、[脆弱性報告](/ja/docs/releases/security-vulnerabilities/#reporting-a-vulnerability)ページに従い、Istio プロダクトセキュリティワーキンググループに機能リクエストを送ってください。

Istio プロダクトセキュリティワーキンググループと連絡を取る前に問題を公開しないでください。セキュリティ脆弱性として非公開で修正すべき場合があります。

セキュリティ脆弱性でないと判断された場合は、公開の場で issue が作成され、議論が進みます。

### 既知の制限 {#known-limitations}

認可ポリシーの既知の制限を以下に示します。

#### サーバーファースト TCP プロトコルはサポートされていません {#server-first-tcp-protocols-are-not-supported}

サーバーファースト TCP プロトコルとは、サーバーアプリが TCP 接続確立直後に最初のバイトを送信し、その後クライアントからデータを受信するプロトコルです。

現状、認可ポリシーはインバウンドトラフィックのみ制御でき、アウトバウンドトラフィックは制御できません。

また、サーバーファースト TCP プロトコルもサポートされません。サーバーはクライアントからデータを受け取る前に最初のバイトを送信するため、この最初のバイトは認可ポリシーのチェックなしにクライアントに返されます。

この最初のバイトに認可で保護すべき機密データが含まれる場合、認可ポリシーは使うべきではありません。

最初のバイトが機密でなければ、たとえば公開データのネゴシエーションなどには認可ポリシーを使えます。最初のバイト以降のリクエストには通常通り認可ポリシーが適用されます。

## トラフィックキャプチャの限界を理解する {#understand-traffic-capture-limitations}

Istio Sidecar の仕組みは、インバウンド・アウトバウンドトラフィックをインターセプトし、Sidecar プロキシに転送します。

ただし、**すべて**のトラフィックがインターセプトされるわけではありません：

- 転送は TCP ベースのトラフィックのみ。UDP や ICMP パケットはインターセプト・変更されません。
- インバウンドインターセプトは多くの [Sidecar 使用ポート](/ja/docs/ops/deployment/application-requirements/#ports-used-by-istio)やポート 22 では無効です。このリストは `traffic.sidecar.istio.io/excludeInboundPorts` などで拡張可能です。
- アウトバウンドインターセプトは `traffic.sidecar.istio.io/excludeOutboundPorts` などの設定や他の方法で無効化できます。

一般に、アプリとその Sidecar プロキシ間のセキュリティ境界は非常に薄いです。Sidecar 設定は Pod 単位で行われ、両者は同じネットワーク／プロセス空間で動作します。したがって、アプリはインターセプトルールを削除したり、Sidecar プロキシを削除・変更・置換できます。これにより、Pod は意図的にアウトバウンドトラフィックを Sidecar 経由せずに送信したり、インバウンドトラフィックを Sidecar を経由せずに受信できます。

したがって、Istio だけにすべてのトラフィックのインターセプトを依存するのは安全ではありません。正しいセキュリティ境界は、クライアントが**他の** Pod の Sidecar をバイパスできないことです。

たとえば、ポート `9080` で `reviews` アプリを動かしている場合、`productpage` アプリからのすべてのトラフィックは `reviews` Sidecar プロキシでインターセプトされ、Istio の認証・認可ポリシーが適用されるべきです。

### `NetworkPolicy` による多層防御 {#defense-in-depth-with-network-policy}

さらなるトラフィックセキュリティのため、Istio ポリシーは Kubernetes の[ネットワークポリシー](https://kubernetes.io/ja/docs/concepts/services-networking/network-policies/)と組み合わせて使えます。これにより、強力な[多層防御](<https://en.wikipedia.org/wiki/Defense_in_depth_(computing)>)が実現できます。

たとえば、`reviews` アプリへのトラフィックはポート `9080` のみ許可するなど。セキュリティ基準を満たさない Pod や脆弱な Pod があっても、攻撃者の行動を制限・阻止できます。

実際の挙動によっては、ネットワークポリシーの変更が Istio プロキシの既存接続に影響しない場合があります。ポリシー適用後は Istio プロキシを再起動し、既存接続を切断して新しいポリシーを適用してください。

### Egress トラフィックのセキュリティ確保 {#securing-egress-traffic}

[`outboundTrafficPolicy: REGISTRY_ONLY`](/ja/docs/tasks/traffic-management/egress/egress-control/#envoy-passthrough-to-external-services) のような設定は、未宣言サービスへのアクセスを防ぐセキュリティポリシーとしては十分ではありません。あくまでベストエフォートです。

意図しない依存を防ぐには有効ですが、Egress トラフィックのセキュリティを本当に確保し、すべてのアウトバウンドトラフィックをプロキシ経由にしたい場合は、[Egress Gateway](/ja/docs/tasks/traffic-management/egress/egress-gateway/) を使いましょう。[ネットワークポリシー](/ja/docs/tasks/traffic-management/egress/egress-gateway/#apply-kubernetes-network-policies)と組み合わせることで、すべてまたは一部のアウトバウンドトラフィックを Egress Gateway 経由に強制できます。これにより、クライアントが意図的または悪意でプロキシをバイパスしてもリクエストはブロックされます。

## TLS オリジネーション時の DestinationRule での TLS 検証設定 {#configure-TLS-verification-in-destination-rule-when-using-TLS-origination}

Istio は Sidecar プロキシやゲートウェイから[TLS オリジネーション](/ja/docs/tasks/traffic-management/egress/egress-tls-origination/)をサポートします。これにより、アプリからのプレーン HTTP トラフィックを透過的に HTTPS に「アップグレード」できます。

`DestinationRule` の `tls` フィールドを設定する際は、`caCertificates`、`subjectAltNames`、`sni` フィールドに注意してください。Istiod で環境変数 `VERIFY_CERTIFICATE_AT_CLIENT=true` を有効にすると、システム証明書ストアの CA 証明書が自動的に `caCertificate` に設定されます。OS CA 証明書が特定ホストにのみ有効な場合は、Istiod で `VERIFY_CERTIFICATE_AT_CLIENT=false` を設定し、`DestinationRule` で `caCertificates` を `system` に設定できます。
`DestinationRule` で `caCertificates` を指定すると、OS CA 証明書より優先されます。
デフォルトでは、Egress トラフィックは TLS ハンドシェイク時に SNI を送信しません。
SNI は `DestinationRule` で明示的に設定してください。

{{< warning >}}
サーバー証明書の検証には、`caCertificates` と `subjectAltNames` の両方が必要です。

CA だけでサーバー証明書を検証するのは不十分で、サブジェクト代替名の検証も必要です。

`VERIFY_CERTIFICATE_AT_CLIENT` を設定しても `subjectAltNames` を設定しなければ、すべての証明書を検証できません。

サーバーが CA 証明書を使っていない場合、`subjectAltNames` の設定有無にかかわらず使われません。
{{< /warning >}}

例：

{{< text yaml >}}
apiVersion: networking.istio.io/v1
kind: DestinationRule
metadata:
name: google-tls
spec:
host: google.com
trafficPolicy:
tls:
mode: SIMPLE
caCertificates: /etc/ssl/certs/ca-certificates.crt
subjectAltNames: - "google.com"
sni: "google.com"
{{< /text >}}

## ゲートウェイ {#gateways}

Istio で[ゲートウェイ](/ja/docs/tasks/traffic-management/ingress/)を運用する場合、以下のリソースが関与します：

- `Gateway` はゲートウェイのポートと TLS 設定を制御します。
- `VirtualService` はルーティングロジックを制御します。VirtualService は `Gateway` リソースの `gateways` フィールドで直接参照され、`Gateway` と `VirtualService` の `hosts` フィールドは一致させる必要があります。

### `Gateway` 作成権限の制限 {#restrict-gateway-creation-privileges}

Istio では、ゲートウェイリソースの作成権限は信頼できるクラスタ管理者のみに付与することを推奨します。これは
[Kubernetes RBAC ポリシー](https://kubernetes.io/ja/docs/reference/access-authn-authz/rbac/)
や [Open Policy Agent](https://www.openpolicyagent.org/) などで実現できます。

### 過度に広い `hosts` 設定を避ける {#avoid-overly-broad-hosts-configurations}

可能であれば、`Gateway` リソースの `hosts` フィールドは過度に広くしないでください。

たとえば、以下の設定は任意の `VirtualService` が `Gateway` にバインドでき、意図しないドメインが公開されるリスクがあります：

{{< text yaml >}}
servers:

- port:
  number: 80
  name: http
  protocol: HTTP
  hosts:
  - "\*"
    {{< /text >}}

このような設定は、特定のドメインや名前空間のみに制限すべきです：

{{< text yaml >}}
servers:

- port:
  number: 80
  name: http
  protocol: HTTP
  hosts:
  - "foo.example.com" # foo.example.com 用の VirtualService のみ許可
  - "default/bar.example.com" # default 名前空間の bar.example.com 用 VirtualService のみ許可
  - "route-namespace/\*" # route-namespace 名前空間の任意ホスト用 VirtualService のみ許可
    {{< /text >}}

### 機密ワークロードの分離 {#isolate-sensitive-services}

機密ワークロードを物理的に厳格に分離したい場合があります。たとえば、機密ドメイン `payments.example.com` を[専用ゲートウェイインスタンス](/ja/docs/setup/install/istioctl/#configure-gateways)で運用し、`blog.example.com` や `store.example.com` などの低機密ドメインは共有ゲートウェイで運用するなどです。これにより多層防御が強化され、規制要件にも対応しやすくなります。

### 緩い SNI マッチで機密 http ホストを明示的に拒否 {#explicitly-disable-all-the-sensitive-http-host-under-relaxed-SNI-host-matching}

複数の `Gateway` リソースで異なるホストに対し双方向・単方向 TLS を設定するのは一般的です。たとえば、SNI ホスト `admin.example.com` で双方向 TLS、SNI ホスト `*.example.com` で単方向 TLS など。

{{< text yaml >}}
kind: Gateway
metadata:
name: guestgateway
spec:
selector:
istio: ingressgateway
servers:

- port:
  number: 443
  name: https
  protocol: HTTPS
  hosts:
  - "\*.example.com"
    tls:
    mode: SIMPLE

---

kind: Gateway
metadata:
name: admingateway
spec:
selector:
istio: ingressgateway
servers:

- port:
  number: 443
  name: https
  protocol: HTTPS
  hosts: - admin.example.com
  tls:
  mode: MUTUAL
  {{< /text >}}

このような設定が必要な場合は、`admin.example.com` の http ホストを `*.example.com` 設定の VirtualService から明示的に除外することを推奨します。現在 [Envoy プロキシは](https://github.com/envoyproxy/envoy/issues/6767) http1 の `Host` や http2 の `:authority` ヘッダが SNI 制約に従うことを要求しません。つまり、攻撃者はゲスト用 SNI TLS 接続を使って管理者用 VirtualService にアクセスできてしまいます。http ステータス 421 は SNI 不一致時の拒否に使えます。

{{< text yaml >}}
apiVersion: networking.istio.io/v1
kind: VirtualService
metadata:
name: disable-sensitive
spec:
hosts:

- "admin.example.com"
  gateways:
- guestgateway
  http:
- match: - uri:
  prefix: /
  fault:
  abort:
  percentage:
  value: 100
  httpStatus: 421
  route: - destination:
  port:
  number: 8000
  host: dest.default.cluster.local
  {{< /text >}}

## プロトコル検出 {#protocol-detection}

Istio は[トラフィックプロトコルを自動判別](/ja/docs/ops/configuration/traffic-management/protocol-selection/#automatic-protocol-selection)できますが、誤検出や意図しない動作を防ぐため、[明示的なプロトコル指定](/ja/docs/ops/configuration/traffic-management/protocol-selection/#explicit-protocol-selection)を推奨します。

## CNI（Container Network Interface） {#CNI}

すべてのトラフィックを透過的にインターセプトするため、Istio は `istio-init` `initContainer` で `iptables` ルールを設定します。これには Pod に `NET_ADMIN` や `NET_RAW` [ケーパビリティ](https://kubernetes.io/ja/docs/tasks/configure-pod-container/security-context/#set-capabilities-for-a-container)が必要です。

Pod への権限付与を減らすため、Istio には [CNI プラグイン](/ja/docs/setup/additional-setup/cni/)があり、これにより上記権限が不要になります。

## 軽量な Docker イメージの利用 {#use-hardened-docker-images}

Istio のデフォルト Docker イメージ（コントロールプレーン、ゲートウェイ、Sidecar プロキシ用）は `ubuntu` ベースです。`bash` や `curl` など多くのツールが含まれ、利便性と攻撃面の広さのトレードオフとなっています。

Istio にはより軽量な [Distroless イメージ](/ja/docs/ops/configuration/security/harden-docker-images/) もあり、依存関係が最小限です。

{{< warning >}}
Distroless イメージは現在 Alpha 機能です。
{{< /warning >}}

## リリースとセキュリティポリシー {#release-and-security-policy}

セキュリティ脆弱性の最新パッチを適用するには、最新の Istio パッチを常に適用し、[サポート対象リリース](/ja/docs/releases/supported-releases)を使ってください。

## 無効な設定の検出 {#detect-invalid-configurations}

Istio はリソース作成時に検証を行いますが、すべての設定問題をカバーできるわけではなく、意図せず無効な設定が適用されない場合があります。

- 設定前後に `istioctl analyze` を実行し、設定の有効性を確認してください。
- コントロールプレーンで拒否された設定がないか監視してください。拒否はログや `pilot_total_xds_rejects` メトリクスで確認できます。
- 設定が期待通り動作するかテストしてください。セキュリティポリシーの場合、正・負両方のテストで過剰・過少な制約がないか確認しましょう。

## Alpha・実験的機能の利用回避 {#avoid-alpha-and-experimental-features}

Istio のすべての機能・API には[機能ステージ](/ja/docs/releases/feature-stages/)が定義されており、安定性・廃止・セキュリティポリシーが明記されています。

Alpha や実験的機能はセキュリティ保証が弱いため、極力利用を避けてください。これらの機能によるセキュリティ問題はすぐに修正されない場合や、標準の[セキュリティ脆弱性](/ja/docs/releases/security-vulnerabilities/)プロセスに従わない場合があります。

利用中の機能のステージは [Istio 機能一覧](/ja/docs/releases/feature-stages/#istio-features)で確認してください。

<!-- 将来的には `istioctl` コマンドで確認できるようにドキュメント化予定 -->

## ロックダウンポート {#lock-down-ports}

Istio は[一連のロックダウンポート](/ja/docs/ops/deployment/application-requirements/#ports-used-by-istio)を設定し、セキュリティを強化しています。

### コントロールプレーン {#control-plane}

Istiod は利便性のため、いくつかの未認証プレーンテキストポートを公開しています。理想的にはこれらのポートは閉じるべきです：

- ポート `8080` はデバッグインターフェースを公開し、クラスタ状態の詳細な読み取り権限を提供します。Istiod で環境変数 `ENABLE_DEBUG_ON_HTTP=false` を設定して無効化できます。
  注意：多くの `istioctl` コマンドはこのインターフェースに依存しており、無効化すると動作しない場合があります。
- ポート `15010` は XDS サービスをプレーンテキストで公開します。Istiod デプロイに `--grpcAddr=""` フラグを追加して無効化できます。
  注意：証明書発行・配布などの高機密サービスは決してプレーンテキストで公開しないでください。

### データプレーン {#data-plane}

プロキシは複数のポートを公開します。外部公開はポート `15090`（テレメトリ）と `15021`（ヘルスチェック）のみです。
ポート `15020` と `15000` はデバッグ用で、`localhost` のみ公開です。つまり、アプリとプロキシは同じ Pod 内で相互アクセスでき、Sidecar とアプリ間に信頼境界はありません。

## サードパーティサービスアカウントトークンの設定 {#configure-third-party-service-account-tokens}

Istio データプレーンの認証にはサービスアカウントトークンを使います。Kubernetes には 2 種類のトークンがあります：

- サードパーティトークン（有効期限・スコープあり）
- ファーストパーティトークン（有効期限なし、すべての Pod にマウント）

ファーストパーティトークンは安全性が低いため、Istio はデフォルトでサードパーティトークンを使いますが、すべての Kubernetes プラットフォームで有効とは限りません。

`istioctl` でインストールする場合は自動検出されますが、`--set values.global.jwtPolicy=third-party-jwt` または `--set values.global.jwtPolicy=first-party-jwt` で手動設定も可能です。

サードパーティトークン対応かどうかは `TokenRequest` API で確認できます。以下のようなレスポンスがなければ未対応です：

{{< text bash >}}
$ kubectl get --raw /api/v1 | jq '.resources[] | select(.name | index("serviceaccounts/token"))'
{
"name": "serviceaccounts/token",
"singularName": "",
"namespaced": true,
"group": "authentication.k8s.io",
"version": "v1",
"kind": "TokenRequest",
"verbs": [
"create"
]
}
{{< /text >}}

多くのクラウドベンダーは対応済みですが、ローカル開発ツールやカスタムインストールでは Kubernetes 1.20 未満も多いです。有効化方法は [Kubernetes ドキュメント](https://kubernetes.io/ja/docs/tasks/configure-pod-container/configure-service-account/#service-account-token-volume-projection) を参照してください。

## 下流接続数制限の設定 {#configure-a-limit-on-downstream-connections}

デフォルトでは、Istio（および Envoy）は下流接続数に制限がありません。これは悪意ある攻撃に悪用される可能性があります（[security bulletin 2020-007](/ja/news/security/istio-security-2020-007/) 参照）。この問題を防ぐには、環境に応じて適切な接続数制限を設定してください。

### `global_downstream_max_connections` 値の設定 {#configure-global_downstream_max_connections-value}

インストール時に以下の設定を追加できます：

{{< text yaml >}}
meshConfig:
defaultConfig:
runtimeValues:
"overload.global_downstream_max_connections": "100000"
{{< /text >}}
