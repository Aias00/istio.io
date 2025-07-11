---
title: L7 機能の利用
description: L7 waypoint プロキシを利用する際にサポートされる機能。
weight: 50
owner: istio/wg-networking-maintainers
test: no
---

トラフィックフローに waypoint プロキシを追加することで、より多くの [Istio の機能](/ja/docs/concepts)を有効化できます。
waypoint は {{< gloss "gateway api" >}}Kubernetes Gateway API{{< /gloss >}} で設定します。

{{< warning >}}
VirtualService と Ambient データプレーンモードの組み合わせは、まだ Alpha 段階です。
Gateway API 設定との混在利用はサポートされておらず、未定義の動作を引き起こす可能性があります。
{{< /warning >}}

{{< warning >}}
`EnvoyFilter` は Envoy プロキシの高度な設定のための Istio のエスケープハッチ API です。
**`EnvoyFilter` は現時点の waypoint プロキシを含む Istio バージョンではサポートされていません**。
限定的なシナリオで waypoint と組み合わせて `EnvoyFilter` を使うことはできますが、現状この API はサポートされておらず、メンテナは強く非推奨としています。Alpha API の進化に伴い、将来のバージョンで問題が発生する可能性があります。公式サポートは今後提供される見込みです。
{{< /warning >}}

## ルーティングとポリシーのアタッチ {#route-and-policy-attachment}

Gateway API では**アタッチメント**によってオブジェクト（ルートやゲートウェイなど）間の関係を定義します。

- ルートオブジェクト（例: [HTTPRoute](https://gateway-api.sigs.k8s.io/api-types/httproute/)）は、アタッチしたい**親**リソースを参照する方法を持ちます。
- ポリシーオブジェクトは [**メタリソース**](https://gateway-api.sigs.k8s.io/geps/gep-713/)とみなされ、標準的な方法で**ターゲット**オブジェクトの挙動を拡張します。

下表は各オブジェクトのアタッチメントタイプを示します。

## トラフィックルーティング {#traffic-routing}

waypoint プロキシをデプロイすると、以下のトラフィックルートタイプが利用できます：

| 名称                                                                | 機能ステータス | アタッチ方法 |
| ------------------------------------------------------------------- | -------------- | ------------ |
| [`HTTPRoute`](https://gateway-api.sigs.k8s.io/guides/http-routing/) | Beta           | `parentRefs` |
| [`TLSRoute`](https://gateway-api.sigs.k8s.io/guides/tls)            | Alpha          | `parentRefs` |
| [`TCPRoute`](https://gateway-api.sigs.k8s.io/guides/tcp/)           | Alpha          | `parentRefs` |

これらのルートで実現できる機能範囲については[トラフィック管理](/ja/docs/tasks/traffic-management/)ドキュメントを参照してください。

## セキュリティ {#security}

waypoint をインストールしない場合は[四層セキュリティポリシー](/ja/docs/ambient/usage/l4-policy/)のみ利用できます。
waypoint を追加することで、以下のポリシーが利用可能になります：

| 名称                                                                                             | 機能ステータス | アタッチ方法 |
| ------------------------------------------------------------------------------------------------ | -------------- | ------------ |
| [`AuthorizationPolicy`](/ja/docs/reference/config/security/authorization-policy/)（L7 機能含む） | Beta           | `targetRefs` |
| [`RequestAuthentication`](/ja/docs/reference/config/security/request_authentication/)            | Beta           | `targetRefs` |

### 認可ポリシーの注意事項 {#considerations}

Ambient モードでは、認可ポリシーは**ターゲット**（ztunnel で実行）または**アタッチ**（waypoint で実行）として機能します。
認可ポリシーを waypoint にアタッチするには、waypoint を参照する `targetRef` を持つか、その waypoint のサービスを利用する必要があります。

ztunnel は L7 ポリシーを強制できません。ワークロードセレクタ（`targetRef` のアタッチではなく）で L7 属性にマッチするルールを持つポリシーをターゲットにした場合、ztunnel による強制は安全上 `DENY` ポリシーに変換されます。

詳細は [L4 ポリシーガイド](/ja/docs/ambient/usage/l4-policy/)を参照してください。
TCP 専用ユースケースでポリシーを waypoint にアタッチするタイミングも含みます。

## オブザーバビリティ {#observability}

[Istio の全トラフィックメトリクス](/ja/docs/reference/config/metrics/)は waypoint プロキシからエクスポートされます。

## 拡張 {#extension}

waypoint プロキシは {{< gloss >}}Envoy{{< /gloss >}} のデプロイメントであり、{{< gloss "sidecar">}}Sidecar モード{{< /gloss >}}の Envoy で利用できる一部の拡張メカニズムも waypoint プロキシで利用できます。

| 名称           | 機能ステータス | アタッチ方法 |
| -------------- | -------------- | ------------ |
| `WasmPlugin` † | Alpha          | `targetRefs` |

† [WebAssembly プラグインで waypoint を拡張する方法の詳細はこちら](/ja/docs/ambient/usage/extend-waypoint-wasm/)。

拡張設定は Gateway API でポリシーとして扱われます。

## ルートやポリシーのスコープ指定 {#scoping-routes-or-policies}

ルートやポリシーは waypoint プロキシを通過するすべてのトラフィック、または特定サービスのみに適用できます。

### waypoint プロキシ全体にアタッチ {#attach-to-the-entire-waypoint-proxy}

ルートやポリシーを waypoint 全体にアタッチするには（それを利用するすべてのトラフィックに適用するには）、タイプに応じて `Gateway` を `parentRefs` または `targetRefs` の値に設定します。

`default` ネームスペースの `default` waypoint に `AuthorizationPolicy` ポリシーを適用する例：

{{< text yaml >}}
apiVersion: security.istio.io/v1
kind: AuthorizationPolicy
metadata:
name: view-only
namespace: default
spec:
targetRefs:

- kind: Gateway
  group: gateway.networking.k8s.io
  name: default
  action: ALLOW
  rules:
- from: - source:
  namespaces: ["default", "istio-system"]
  to: - operation:
  methods: ["GET"]
  {{< /text >}}

### 特定サービスにアタッチ {#attach-to-a-specific-service}

ルートを waypoint 内の 1 つまたは複数の特定サービスにアタッチすることもできます。
必要に応じて `Service` を `parentRefs` または `targetRefs` の値に設定します。

`default` ネームスペースの `reviews` サービスに `reviews` HTTPRoute を適用する例：

{{< text yaml >}}
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
name: reviews
namespace: default
spec:
parentRefs:

- group: ""
  kind: Service
  name: reviews
  port: 9080
  rules:
- backendRefs: - name: reviews-v1
  port: 9080
  weight: 90 - name: reviews-v2
  port: 9080
  weight: 10
  {{< /text >}}
