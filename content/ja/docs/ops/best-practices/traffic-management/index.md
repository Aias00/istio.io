---
title: トラフィック管理のベストプラクティス
description: ネットワークやトラフィック管理の問題を回避するための設定ベストプラクティス。
force_inline_toc: true
weight: 20
aliases:
  - /zh/help/ops/traffic-management/deploy-guidelines
  - /zh/help/ops/deploy-guidelines
  - /zh/docs/ops/traffic-management/deploy-guidelines
owner: istio/wg-networking-maintainers
test: n/a
---

このセクションでは、ネットワークやトラフィック管理の問題を回避するための、特定のデプロイや設定ガイドラインを提供します。

## サービスにデフォルトルートを設定する {#set-default-routes-for-services}

Istio のデフォルト動作では、ルールを何も設定しなくても、あらゆる発信元からのトラフィックが対象サービスのすべてのバージョンに送信されます。
しかし、Istio でのベストプラクティスは、最初から各サービスにデフォルトルートを持つ `VirtualService` を作成することです。

最初はサービスが 1 バージョンしかなくても、2 つ目のバージョンをデプロイしたい場合、
新バージョンが無制御にトラフィックを受け取るのを防ぐため、**新バージョンを有効化する前に**ルートルールを設定しておく必要があります。

Istio のデフォルトのラウンドロビンルーティングに依存するもう 1 つの潜在的な問題は、
Istio の `DestinationRule` 評価アルゴリズムの微妙な挙動にあります。
リクエストのルーティング時、Envoy はまず `VirtualService` のルートルールを評価し、特定のサブセットにルーティングするかどうかを決定します。
このときのみ、そのサブセットに対応する `DestinationRule` ポリシーが有効になります。
したがって、トラフィックを**明示的に**該当サブセットにルーティングした場合のみ、Istio はそのサブセット用に定義したポリシーを適用します。

たとえば、以下の `DestinationRule` が **reviews** サービスの唯一の設定で、
対応する `VirtualService` にはルートルールがないとします：

{{< text yaml >}}
apiVersion: networking.istio.io/v1
kind: DestinationRule
metadata:
name: reviews
spec:
host: reviews
subsets:

- name: v1
  labels:
  version: v1
  trafficPolicy:
  connectionPool:
  tcp:
  maxConnections: 100
  {{< /text >}}

Istio のデフォルトのラウンドロビンルーティングが `v1` インスタンスを呼び出すことがあっても、
`v1` が唯一の稼働バージョンであっても、上記のトラフィックポリシーは決して適用されません。

この例を修正するには、2 つの方法があります。1 つは `DestinationRule` でトラフィックポリシーを上位に移動し、すべてのバージョンに適用する方法です：

{{< text yaml >}}
apiVersion: networking.istio.io/v1
kind: DestinationRule
metadata:
name: reviews
spec:
host: reviews
trafficPolicy:
connectionPool:
tcp:
maxConnections: 100
subsets:

- name: v1
  labels:
  version: v1
  {{< /text >}}

より良い方法は、`VirtualService` でサービスに適切なルートルールを定義することです。
たとえば、`reviews:v1` へのシンプルなルートルール：

{{< text yaml >}}
apiVersion: networking.istio.io/v1
kind: VirtualService
metadata:
name: reviews
spec:
hosts:

- reviews
  http:
- route: - destination:
  host: reviews
  subset: v1
  {{< /text >}}

## 設定の名前空間間共有の制御 {#cross-namespace-configuration}

1 つの名前空間で `VirtualService`、`DestinationRule`、`ServiceEntry` を定義し、
それらを他の名前空間にエクスポートして再利用できます。
Istio はデフォルトですべてのトラフィック管理リソースを全名前空間にエクスポートしますが、
`exportTo` を使って名前空間間の可視性を制御できます。たとえば、同じ名前空間のクライアントだけが以下の `VirtualService` を利用できます：

{{< text yaml >}}
apiVersion: networking.istio.io/v1
kind: VirtualService
metadata:
name: myservice
spec:
hosts:

- myservice.com
  exportTo:
- "."
  http:
- route: - destination:
  host: myservice
  {{< /text >}}

{{< tip >}}
Kubernetes の `Service` も `networking.istio.io/exportTo` アノテーションで可視性を制御できます。
{{< /tip >}}

特定の名前空間で `DestinationRule` の可視性を設定しても、そのルールが必ず使われるとは限りません。
`DestinationRule` を他の名前空間にエクスポートすると、その名前空間で利用できますが、
実際にリクエスト時にその `DestinationRule` が適用されるには、名前空間が `DestinationRule` の検索パス上にある必要があります：

1. クライアントの名前空間
1. サービスの名前空間
1. Istio ルート設定の名前空間（デフォルトは `istio-system`）

たとえば、以下の `DestinationRule`：

{{< text yaml >}}
apiVersion: networking.istio.io/v1
kind: DestinationRule
metadata:
name: myservice
spec:
host: myservice.default.svc.cluster.local
trafficPolicy:
connectionPool:
tcp:
maxConnections: 100
{{< /text >}}

これを `ns1` 名前空間で作成したとします。

`ns1` のクライアントから `myservice` サービスにリクエストすると、
検索パスの最初（クライアントの名前空間）にあるため、この `DestinationRule` が適用されます。

今度は別の名前空間（例：`ns2`）からリクエストすると、クライアントは `ns1` とは異なる名前空間にいるため、
`DestinationRule` は検索パスのどこにも存在しません。
サービス `myservice.default.svc.cluster.local` も `ns1` ではなく `default` 名前空間にあるため、
検索パスの 2 番目（サービスの名前空間）にも `DestinationRule` はありません。

たとえ `myservice` サービスを全名前空間にエクスポートし、`DestinationRule` も全名前空間にエクスポートしても、
`ns2` からのリクエストにはこのルールは適用されません。検索パス上にないからです。

この問題を回避するには、該当サービスと同じ名前空間（この例では `default`）で `DestinationRule` を作成します。
これで、どの名前空間のクライアントからのリクエストにも適用されます。
または、`DestinationRule` を `istio-system` 名前空間に移動することもできますが、
これは全名前空間に適用するグローバル設定の場合のみ推奨され、管理者権限が必要です。

Istio がこのような制限付きの `DestinationRule` 検索パスを採用している理由は 2 つあります：

1. 無関係な名前空間のサービス動作を上書きする `DestinationRule` の定義を防ぐため。
1. 同じ host に複数の `DestinationRule` がある場合、明確な検索順序を持たせるため。

## 大きな `VirtualService` や `DestinationRule` を複数リソースに分割する {#split-virtual-services}

特定の host のルートルールやポリシーセットを 1 つの `VirtualService` や `DestinationRule` で定義するのが困難な場合、
複数のリソースで段階的に host の設定を指定するのがベストです。
これらの `DestinationRule` をゲートウェイにバインドすると、コントロールプレーンはそれらの `DestinationRule` や `VirtualService` をマージします。

たとえば、1 つの `VirtualService` が Ingress Gateway にバインドされ、
host に基づいて複数のサービスにパスベースでプロキシしている場合：

{{< text yaml >}}
apiVersion: networking.istio.io/v1
kind: VirtualService
metadata:
name: myapp
spec:
hosts:

- myapp.com
  gateways:
- myapp-gateway
  http:
- match:
  - uri:
    prefix: /service1
    route:
  - destination:
    host: service1.default.svc.cluster.local
- match:
  - uri:
    prefix: /service2
    route:
  - destination:
    host: service2.default.svc.cluster.local
- match:
  ...
  {{< /text >}}

この構成の欠点は、下層のマイクロサービスの他の設定（ルートルールなど）もこの設定ファイルに含める必要があり、
各サービスチームが所有する個別リソースに分離できないことです。詳細は[Ingress Gateway リクエストでルートルールが効かない](/ja/docs/ops/common-problems/network-issues/#route-rules-have-no-effect-on-ingress-gateway-requests)を参照してください。

この問題を回避するには、`myapp.com` の設定を各バックエンドサービスごとに複数の `VirtualService` に分割するのがベストです。例：

{{< text yaml >}}
apiVersion: networking.istio.io/v1
kind: VirtualService
metadata:
name: myapp-service1
spec:
hosts:

- myapp.com
  gateways:
- myapp-gateway
  http:
- match:
  - uri:
    prefix: /service1
    route:
  - destination:
    host: service1.default.svc.cluster.local

---

apiVersion: networking.istio.io/v1
kind: VirtualService
metadata:
name: myapp-service2
spec:
hosts:

- myapp.com
  gateways:
- myapp-gateway
  http:
- match:
  - uri:
    prefix: /service2
    route:
  - destination:
    host: service2.default.svc.cluster.local

---

apiVersion: networking.istio.io/v1
kind: VirtualService
metadata:
name: myapp-...
{{< /text >}}

既存の host に 2 つ目以降の `VirtualService` を作成すると、`istiod` は追加のルートルールを host の既存設定にマージします。
ただし、この機能を使う際には注意点があります。

1. 各 `VirtualService` 内のルール評価順序は保持されますが、リソース間の順序は不定です。
   つまり、分割設定間でルールの順序依存や競合がある場合、予測できない動作になります。
1. 各分割設定には `catch-all` ルール（すべてのリクエストパスやヘッダにマッチするルール）は 1 つだけにしてください。
   すべての `catch-all` ルールはマージ後リストの末尾に移動しますが、最初に適用されたものが他を上書きします。
1. `VirtualService` をゲートウェイにバインドする場合のみ分割が可能です。Sidecar では host のマージはサポートされません。

同様のマージ動作と制限は `DestinationRule` の分割にも適用されます。

1. 同じ host の複数の `DestinationRule` で、任意のサブセットは 1 つだけ定義してください。複数ある場合は最初の定義が使われ、以降は無視されます。サブセット内容のマージはサポートされません。
1. 同じ host にはトップレベルの `trafficPolicy` は 1 つだけです。複数の `DestinationRule` で定義した場合、最初のものが使われ、以降は無視されます。
1. `VirtualService` のマージと異なり、`DestinationRule` のマージは Sidecar と gateway の両方で有効です。

## サービスルート再設定時の 503 エラー回避 {#avoid-5-0-3-errors-while-reconfiguring-service-routes}

トラフィックをサービスの特定バージョン（サブセット）にルーティングするルールを設定する際は、
ルートで使うサブセットが事前に利用可能であることを必ず確認してください。
そうしないと、再設定中にサービス呼び出しが 503 エラーになることがあります。

`kubectl` の 1 回のコマンド（例：`kubectl apply -f myVirtualServiceAndDestinationRule.yaml`）で
該当サブセットを定義する `VirtualService` と `DestinationRule` を作成しても不十分です。
なぜなら、リソースは（Kubernetes API サーバーから）最終的整合性で istiod に伝播されるためです。
`VirtualService` がサブセットを参照する前に `DestinationRule` が到達していない場合、
istiod が生成する Envoy 設定は存在しない上流プールを参照し、HTTP 503 エラーが発生します。
すべての設定オブジェクトが istiod で利用可能になるまで、このエラーは続きます。

サブセット付きルートの設定時にダウンタイムゼロを保証するには、以下の「先付け後外し」手順に従ってください：

- 新しいサブセットを追加する場合：

  1. まず `DestinationRule` を更新して新しいサブセットを追加し、その後それを使うすべての `VirtualService` を更新し、`kubectl` などで適用します。

  1. 数秒待ち、`DestinationRule` 設定が Envoy Sidecar に伝播するのを待ちます。

  1. `VirtualService` を更新して新しいサブセットを参照させます。

- サブセットを削除する場合：

  1. `DestinationRule` からサブセットを削除する前に、`VirtualService` からそのサブセットへの参照をすべて削除します。

  1. 数秒待ち、`VirtualService` 設定が Envoy Sidecar に伝播するのを待ちます。

  1. 未使用のサブセットを `DestinationRule` から削除します。
