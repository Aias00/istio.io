---
title: L4 セキュリティポリシーの利用
description: L4 セキュアカバレッジのみを利用する場合にサポートされるセキュリティ機能。
weight: 20
owner: istio/wg-networking-maintainers
test: no
---

Istio の[セキュリティポリシー](/ja/docs/concepts/security)の L4（レイヤー 4）機能は、{{< gloss >}}ztunnel{{< /gloss >}} によって提供され、これらの L4 機能は{{< gloss "ambient" >}}Ambient モード{{< /gloss >}}で利用できます。クラスタに[Kubernetes NetworkPolicy](https://kubernetes.io/ja/docs/concepts/services-networking/network-policies/)をサポートする{{< gloss >}}CNI{{< /gloss >}}プラグインがある場合、これらのポリシーも引き続き利用でき、多層防御に役立ちます。

ztunnel と {{< gloss "waypoint" >}}waypoint プロキシ{{< /gloss >}}の階層構造により、特定のワークロードに対して L7（レイヤー 7）処理を有効化するかどうかを選択できます。
L7 ポリシーや Istio のトラフィックルーティング機能を利用するには、ワークロードに対して[waypoint をデプロイ](/ja/docs/ambient/usage/waypoint)してください。
ポリシーが 2 箇所で強制される可能性があるため、いくつかの[注意事項](#considerations)を理解しておく必要があります。

## ztunnel でのポリシー強制 {#policy-enforcement-using-ztunnel}

ワークロードが{{< gloss "Secure L4 Overlay" >}}セキュアカバレッジモード{{< /gloss >}}に登録されている場合、ztunnel プロキシは認可ポリシーを強制できます。強制ポイントは、接続経路上の受信（サーバー側）ztunnel プロキシです。

基本的な L4 認可ポリシーの例：

{{< text yaml >}}
apiVersion: security.istio.io/v1
kind: AuthorizationPolicy
metadata:
name: allow-curl-to-httpbin
spec:
selector:
matchLabels:
app: httpbin
action: ALLOW
rules:

- from:
  - source:
    principals: - cluster.local/ns/ambient-demo/sa/curl
    {{< /text >}}

このポリシーは {{< gloss "sidecar" >}}Sidecar モード{{< /gloss >}}でも Ambient モードでも利用できます。

Istio の `AuthorizationPolicy` API の L4（TCP）機能は、Ambient モードでも Sidecar モードでも同じ動作をします。
認可ポリシーが設定されていない場合、デフォルト動作は `ALLOW` です。ポリシーが設定されると、そのポリシーがターゲットとする Pod には明示的に許可されたトラフィックのみが許可されます。
上記の例では、`app: httpbin` ラベルを持つ Pod には、`cluster.local/ns/ambient-demo/sa/curl` というプリンシパルからのトラフィックのみが許可され、他のすべてのソースからのトラフィックは拒否されます。

## ポリシーのターゲット指定 {#targeting-policies}

Sidecar モードと Ambient モードの L4 ポリシーは、**ターゲット指定**の方法が同じです：
ポリシーのスコープは、そのポリシーオブジェクトが存在するネームスペースと、`spec` 内のオプションの `selector` で決まります。
ポリシーが Istio ルートネームスペース（通常は `istio-system`）にある場合、そのポリシーはすべてのネームスペースに適用されます。
それ以外のネームスペースにある場合は、そのネームスペース内のみに適用されます。

Ambient モードの L7 ポリシーは、{{< gloss "gateway api" >}}Kubernetes Gateway API{{< /gloss >}}で設定された waypoint によって強制されます。これらの waypoint は `targetRef` フィールドで**アタッチ**されます。

## 許可されるポリシー属性 {#allowed-policy-attributes}

認可ポリシールールには [source](/ja/docs/reference/config/security/authorization-policy/#Source)（`from`）、[operation](/ja/docs/reference/config/security/authorization-policy/#Operation)（`to`）、[condition](/ja/docs/reference/config/security/authorization-policy/#Condition)（`when`）などの句を含めることができます。

以下の属性リストは、ポリシーが L4 のみを対象とするかどうかを決定します：

| 種類      | 属性             | 正方向マッチ       | 逆方向マッチ    |
| --------- | ---------------- | ------------------ | --------------- |
| Source    | Peer identity    | `principals`       | `notPrincipals` |
| Source    | Namespace        | `namespaces`       | `notNamespaces` |
| Source    | IP block         | `ipBlocks`         | `notIpBlocks`   |
| Operation | Destination port | `ports`            | `notPorts`      |
| Condition | Source IP        | `source.ip`        | 該当なし        |
| Condition | Source namespace | `source.namespace` | 該当なし        |
| Condition | Source identity  | `source.principal` | 該当なし        |
| Condition | Remote IP        | `destination.ip`   | 該当なし        |
| Condition | Remote port      | `destination.port` | 該当なし        |

### L7 条件を含むポリシー {#policies-with-layer-7-conditions}

ztunnel は L7 ポリシーを強制できません。もしポリシーのルールに L7 属性（上記表にない属性）が含まれており、そのポリシーがターゲットとなった場合、そのポリシーは受信側 ztunnel で強制されますが、安全上 `DENY` ポリシーに変換されます。

以下は HTTP GET メソッドのチェックを追加した例です：

{{< text yaml >}}
apiVersion: security.istio.io/v1
kind: AuthorizationPolicy
metadata:
name: allow-curl-to-httpbin
spec:
selector:
matchLabels:
app: httpbin
action: ALLOW
rules:

- from:
  - source:
    principals: - cluster.local/ns/ambient-demo/sa/curl
    to:
  - operation:
    methods: ["GET"]
    {{< /text >}}

クライアント Pod のアイデンティティが正しくても、L7 属性が存在する場合は ztunnel によって接続が拒否されます：

{{< text plain >}}
command terminated with exit code 56
{{< /text >}}

## waypoint 導入時の強制ポイント選択の注意 {#considerations}

waypoint プロキシをワークロードに追加すると、L4 ポリシーを強制できる場所が 2 箇所になります。
（L7 ポリシーは waypoint プロキシでのみ強制されます。）

セキュアカバレッジのみを利用する場合、トラフィックはターゲット ztunnel で**ソース**ワークロードのアイデンティティとして現れます。

waypoint プロキシはソースワークロードのアイデンティティを偽装しません。waypoint がトラフィックパスに入ると、ターゲット ztunnel には**waypoint** のアイデンティティでトラフィックが見えるようになります。

つまり、waypoint を導入した場合、**ポリシー強制の理想的な場所が変化します**。
L4 属性のみを強制したい場合でも、ソースアイデンティティに依存する場合はポリシーを waypoint プロキシにアタッチすべきです。
ターゲットワークロードに対しては、ターゲット ztunnel で「メッシュ内トラフィックは自分の waypoint からのみ受け入れる」などのポリシーを設定できます。

## ピア認証 {#peer-authentication}

Istio の [ピア認証ポリシー](/ja/docs/concepts/security/#peer-authentication)は、双方向 TLS（mTLS）モードの設定に利用でき、ztunnel でサポートされています。

Ambient モードのデフォルトポリシーは `PERMISSIVE` で、Pod はメッシュ内からの mTLS 暗号化トラフィックと外部からの平文トラフィックの両方を受け入れます。`STRICT` モードを有効にすると、Pod は mTLS 暗号化トラフィックのみを受け入れます。

ztunnel と {{< gloss >}}HBONE{{< /gloss >}} は暗黙的に mTLS を利用するため、`DISABLE` モードは利用できません。このようなポリシーは無視されます。
