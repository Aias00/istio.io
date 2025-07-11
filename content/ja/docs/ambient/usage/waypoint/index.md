---
title: waypoint プロキシの設定
description: オプションの Layer 7 プロキシで Istio の全機能を利用する。
weight: 30
aliases:
  - /zh/docs/ops/ambient/usage/waypoint
  - /zh/latest/docs/ops/ambient/usage/waypoint
owner: istio/wg-networking-maintainers
test: yes
---

**waypoint プロキシ**は、Envoy プロキシをベースとしたオプションのデプロイメントで、定義されたワークロード群に Layer 7（L7）処理を追加します。

waypoint プロキシのインストール、アップグレード、スケールはアプリケーションから独立しており、アプリケーション所有者はその存在を意識する必要がありません。
各ワークロードごとに Envoy プロキシインスタンスを並列で動かす Sidecar {{< gloss "data plane" >}}データプレーン{{< /gloss >}}モードと比べて、必要なプロキシ数を大幅に削減できます。

1 つまたは複数の waypoint は、同じセキュリティ境界を持つアプリケーション間で共有できます。
これは特定ワークロードの全インスタンスや、ネームスペース内の全ワークロードなどが該当します。

{{< gloss "sidecar" >}}Sidecar{{< /gloss >}} モードと比べて、Ambient モードではポリシーは**宛先**の waypoint で強制されます。多くの場合、waypoint はリソース（ネームスペース、サービス、Pod）のゲートウェイの役割を果たします。
Istio はリソースへの全てのトラフィックを waypoint 経由で通し、waypoint がそのリソースの全ポリシーを強制します。

## waypoint プロキシは必要ですか？ {#do-you-need-a-waypoint-proxy}

Ambient の階層的アプローチにより、ユーザーは Istio をより細かく段階的に導入できます。
メッシュなしから安全な L4 カバレッジ、さらに完全な L7 処理へとスムーズに移行できます。

Ambient モードの多くの機能は ztunnel ノードプロキシによって提供されます。
ztunnel は Layer 4（L4）トラフィックのみを扱うため、共有コンポーネントとして安全に動作します。

リダイレクト先として waypoint を設定すると、ztunnel がその waypoint へトラフィックを転送します。
アプリケーションで以下のいずれかの L7 メッシュ機能が必要な場合は、waypoint プロキシが必要です：

- **トラフィック管理**：HTTP ルーティングとロードバランシング、サーキットブレーカー、レートリミット、フォールトインジェクション、リトライ、タイムアウト
- **セキュリティ**：リクエストタイプや HTTP ヘッダーなど L7 属性に基づく高度な認可ポリシー
- **可観測性**：HTTP メトリクス、アクセスログ、分散トレーシング

## waypoint プロキシのデプロイ {#deploy-a-waypoint-proxy}

waypoint プロキシは Kubernetes Gateway リソースとしてデプロイします。

{{< boilerplate gateway-api-install-crds >}}

istioctl の waypoint サブコマンドで、これらのリソースの生成・適用・一覧表示が可能です。

waypoint をデプロイした後、ネームスペース全体（または任意のサービスや Pod）は[登録](#useawaypoint)が必要です。

特定のネームスペースに waypoint プロキシをデプロイする前に、そのネームスペースに `istio.io/dataplane-mode: ambient` ラベルが付与されていることを確認してください：

{{< text syntax=bash snip_id=check_ns_label >}}
$ kubectl get ns -L istio.io/dataplane-mode
NAME STATUS AGE DATAPLANE-MODE
istio-system Active 24h
default Active 24h ambient
{{< /text >}}

`istioctl` で waypoint プロキシ用の Kubernetes Gateway リソースを生成できます。
例えば、`default` ネームスペースで `waypoint` という名前の waypoint プロキシを生成し、そのネームスペース内のサービスのトラフィックを処理する場合：

{{< text syntax=bash snip_id=gen_waypoint_resource >}}
$ istioctl waypoint generate --for service -n default
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
labels:
istio.io/waypoint-for: service
name: waypoint
namespace: default
spec:
gatewayClassName: istio-waypoint
listeners:

- name: mesh
  port: 15008
  protocol: HBONE
  {{< /text >}}

Gateway リソースの `gatewayClassName` は `istio-waypoint` で、Istio 管理の waypoint をインスタンス化します。
`istio.io/waypoint-for: service` ラベルは waypoint がサービスのトラフィックを処理することを示し、これがデフォルトです。

waypoint プロキシを直接デプロイするには、`generate` の代わりに `apply` を使います：

{{< text syntax=bash snip_id=apply_waypoint >}}
$ istioctl waypoint apply -n default
waypoint default/waypoint applied
{{< /text >}}

または、生成した Gateway リソースをデプロイすることもできます：

{{< text syntax=bash >}}
$ kubectl apply -f - <<EOF
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
labels:
istio.io/waypoint-for: service
name: waypoint
namespace: default
spec:
gatewayClassName: istio-waypoint
listeners:

- name: mesh
  port: 15008
  protocol: HBONE
  EOF
  {{< /text >}}

Gateway リソースが適用されると、istiod がリソースを監視し、対応する waypoint Deployment とサービスを自動的にデプロイ・管理します。

### waypoint のトラフィックタイプ {#waypoint-traffic-types}

デフォルトでは、waypoint はそのネームスペース内の**サービス**宛てのトラフィックのみを処理します。
この選択は、Pod 宛てのトラフィックは稀であり、通常は Prometheus のスクレイプなど内部用途で使われ、L7 処理によるオーバーヘッドが不要なためです。

waypoint は全トラフィック、Pod や VM など**ワークロード**宛てのトラフィックのみ、あるいは一切のトラフィックを処理しないようにも設定できます。waypoint へのリダイレクト対象は、`Gateway` オブジェクトの `istio.io/waypoint-for` ラベルで決まります。

`istioctl waypoint apply` の `--for` パラメータで waypoint へのリダイレクト対象を変更できます：

| `waypoint-for` の値 | 宛先タイプ             |
| ------------------- | ---------------------- |
| `service`           | Kubernetes サービス    |
| `workload`          | Pod IP または VM IP    |
| `all`               | サービスとワークロード |
| `none`              | なし（テスト用）       |

waypoint の選択は、トラフィックが**最初に送信**された宛先タイプ（`service` または `workload`）に基づきます。waypoint のないサービス宛てのトラフィックは waypoint へ転送されません。最終的な宛先が waypoint を持つワークロードであっても同様です。

## waypoint プロキシの利用 {#useawaypoint}

waypoint プロキシをデプロイしても、明示的にリソースを設定しない限り、デフォルトではどのリソースにも使われません。

ネームスペース・サービス・Pod で waypoint を利用するには、waypoint 名を値とする `istio.io/use-waypoint` ラベルを追加します。

{{< tip >}}
多くのユーザーはネームスペース全体で waypoint を利用したいと考えるため、まずこの方法を推奨します。
{{< /tip >}}

`istioctl` でネームスペースの waypoint をデプロイする場合、`--enroll-namespace` パラメータで自動的にネームスペースへラベル付与できます：

{{< text syntax=bash snip_id=enroll_ns_waypoint >}}
$ istioctl waypoint apply -n default --enroll-namespace
waypoint default/waypoint applied
namespace default labeled with "istio.io/use-waypoint: waypoint"
{{< /text >}}

または、`kubectl` で `istio.io/use-waypoint: waypoint` ラベルを `default` ネームスペースに追加します：

{{< text syntax=bash >}}
$ kubectl label ns default istio.io/use-waypoint=waypoint
namespace/default labeled
{{< /text >}}

ネームスペースが waypoint 利用として登録されると、Ambient データプレーンモードの Pod からそのネームスペース内のサービスへの全リクエストは waypoint 経由で L7 処理・ポリシー適用されます。

より細かい制御が必要な場合、ネームスペース全体ではなく特定サービスや Pod のみを waypoint 利用に登録できます。ネームスペース内の一部サービスだけ L7 機能が必要な場合や、`WasmPlugin` などの拡張を特定サービスにのみ適用したい場合、または Pod IP 宛てに Kubernetes の[ヘッドレスサービス](https://kubernetes.io/ja/docs/concepts/services-networking/service/#headless-services)を呼び出す場合などに有用です。

{{< tip >}}
ネームスペースとサービスの両方に `istio.io/use-waypoint` ラベルがある場合、サービスの waypoint が `service` または `all` のトラフィックを処理できれば、サービスの waypoint が優先されます。同様に、Pod のラベルはネームスペースより優先されます。
{{< /tip >}}

### サービスを特定の waypoint で利用する設定 {#configure-a-service-to-use-a-specific-waypoint}

例として [Bookinfo アプリ](/ja/docs/examples/bookinfo/) の `reviews` サービスで、`reviews-svc-waypoint` という waypoint をデプロイします：

{{< text syntax=bash >}}
$ istioctl waypoint apply -n default --name reviews-svc-waypoint
waypoint default/reviews-svc-waypoint applied
{{< /text >}}

`reviews` サービスに `reviews-svc-waypoint` waypoint を利用するラベルを付与します：

{{< text syntax=bash >}}
$ kubectl label service reviews istio.io/use-waypoint=reviews-svc-waypoint
service/reviews labeled
{{< /text >}}

メッシュ内の Pod から `reviews` サービスへの全リクエストは、
`reviews-svc-waypoint` waypoint 経由でルーティングされます。

### Pod を特定の waypoint で利用する設定 {#configure-a-pod-to-use-a-specific-waypoint}

`reviews-v2` Pod 用に `reviews-v2-pod-waypoint` という waypoint をデプロイします。

{{< tip >}}
waypoint はデフォルトでサービス宛てです。Pod 宛てにしたい場合は `istio.io/waypoint-for: workload` ラベルが必要で、istioctl の `--for workload` パラメータで生成できます。
{{< /tip >}}

{{< text syntax=bash >}}
$ istioctl waypoint apply -n default --name reviews-v2-pod-waypoint --for workload
waypoint default/reviews-v2-pod-waypoint applied
{{< /text >}}

`reviews-v2` Pod に `reviews-v2-pod-waypoint` waypoint を利用するラベルを付与します：

{{< text syntax=bash >}}
$ kubectl label pod -l version=v2,app=reviews istio.io/use-waypoint=reviews-v2-pod-waypoint
pod/reviews-v2-5b667bcbf8-spnnh labeled
{{< /text >}}

Ambient メッシュ内の Pod から `reviews-v2` Pod IP への全リクエストは、
`reviews-v2-pod-waypoint` waypoint 経由で L7 処理・ポリシー適用されます。

{{< tip >}}
トラフィックの元の宛先タイプでサービスかワークロードの waypoint を使うか決まります。
この方式により、サービスとワークロード両方に waypoint が付与されていても、トラフィックが 2 回 waypoint を通ることはありません。たとえば、最終的に Pod IP へ到達する場合でも、サービス宛てのトラフィックはサービスの waypoint のみを通ります。
{{< /tip >}}

## ネームスペースをまたいだ waypoint の利用 {#usewaypointnamespace}

デフォルトで waypoint プロキシは同じネームスペース内のリソースで利用できます。
Istio 1.23 以降、異なるネームスペースの waypoint も利用可能です。
このセクションでは、クロスネームスペース利用に必要な Gateway 設定と、他ネームスペースの waypoint を利用するリソースの設定方法を説明します。

### クロスネームスペース利用用 waypoint の設定 {#configure-a-waypoint-for-cross-namespace-use}

waypoint を他のネームスペースから利用できるようにするには、
`Gateway` で[他ネームスペースからのルートを許可](https://gateway-api.sigs.k8s.io/reference/spec/#gateway.networking.k8s.io%2fv1.AllowedRoutes)する必要があります。

{{< tip >}}
`allowedRoutes.namespaces.from` に `All` を指定すると、どのネームスペースからもルートを許可できます。
{{< /tip >}}

以下の `Gateway` は、`cross-namespace-waypoint-consumer` というネームスペースのリソースがこの `egress-gateway` を利用できるようにします：

{{< text syntax=yaml >}}
kind: Gateway
metadata:
name: egress-gateway
namespace: common-infrastructure
spec:
gatewayClassName: istio-waypoint
listeners:

- name: mesh
  port: 15008
  protocol: HBONE
  allowedRoutes:
  namespaces:
  from: Selector
  selector:
  matchLabels:
  kubernetes.io/metadata.name: cross-namespace-waypoint-consumer
  {{< /text >}}

### 他ネームスペースの waypoint プロキシを利用するリソースの設定 {#configure-resources-to-use-a-cross-namespace-waypoint-proxy}

デフォルトでは、Istio コントロールプレーンは `istio.io/use-waypoint` ラベルで指定された waypoint を、ラベルを付与したリソースと同じネームスペースで探します。他のネームスペースの waypoint を利用するには、新しいラベル `istio.io/use-waypoint-namespace` を追加します。
`istio.io/use-waypoint-namespace` は `istio.io/use-waypoint` ラベルが使える全リソースで利用できます。
この 2 つのラベルで waypoint の名前とネームスペースを指定します。たとえば、`istio-site` という `ServiceEntry` で、`common-infrastructure` ネームスペースの `egress-gateway` waypoint を利用するには：

{{< text syntax=bash >}}
$ kubectl label serviceentries.networking.istio.io istio-site istio.io/use-waypoint=egress-gateway
serviceentries.networking.istio.io/istio-site labeled
$ kubectl label serviceentries.networking.istio.io istio-site istio.io/use-waypoint-namespace=common-infrastructure
serviceentries.networking.istio.io/istio-site labeled
{{< /text >}}

### クリーンアップ {#cleaning-up}

以下のコマンドでネームスペースからすべての waypoint を削除できます：

{{< text syntax=bash snip_id=delete_waypoint >}}
$ istioctl waypoint delete --all -n default
$ kubectl label ns default istio.io/use-waypoint-
{{< /text >}}

{{< boilerplate gateway-api-remove-crds >}}
