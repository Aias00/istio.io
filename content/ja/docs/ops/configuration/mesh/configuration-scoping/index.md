---
title: 設定スコープ
description: Istio で設定スコープを決定し、運用やパフォーマンス上の利点を得る方法を紹介します。
weight: 60
keywords: [scalability]
owner: istio/wg-networking-maintainers
test: no
---

サービスメッシュをプログラムするために、Istio コントロールプレーン（Istiod）はさまざまな設定を読み込みます。
これには `Service` や `Node` などの Kubernetes のコア型や、`Gateway` などの Istio 独自型が含まれます。
これらはデータプレーンに送信されます（詳細は[アーキテクチャ](/ja/docs/ops/deployment/architecture/)を参照）。

デフォルトでは、コントロールプレーンはすべての名前空間のすべての設定を読み込みます。
各プロキシインスタンスもすべての名前空間の設定を受け取ります。これにはメッシュに登録されていないワークロードの情報も含まれます。

このデフォルトは、すぐに正しく動作することを保証しますが、スケーラビリティのコストが伴います。
各設定には（主に CPU とメモリの）維持・更新コストがかかります。
大規模な場合、リソース消費を抑えるために設定スコープを制限することが重要です。

## スコープの仕組み {#scoping-mechanisms}

Istio には、さまざまなユースケースに合わせて設定スコープを制御するためのツールがいくつか用意されています。
要件に応じて、これらのスコープは単独または組み合わせて使用できます。

- `Sidecar` は特定のワークロードに**インポート**する設定セットを指定する仕組みを提供します
- `exportTo` は設定を**エクスポート**するワークロードセットを指定する仕組みを提供します
- `discoverySelectors` は Istio が特定の設定を完全に無視する仕組みを提供します

### `Sidecar` インポート {#sidecar-import}

`Sidecar` の [`egress.hosts`](/ja/docs/reference/config/networking/sidecar/#IstioEgressListener) フィールドで、インポートする設定リストを指定できます。
`Sidecar` リソースの影響を受ける Sidecar は、指定条件に合致する設定のみを参照できます。

例：

{{< text yaml >}}
apiVersion: networking.istio.io/idecar
metadata:
name: default
spec:
egress:

- hosts: - "./_" # 自分の名前空間からすべての設定をインポート - "bookinfo/_" # bookinfo 名前空間からすべての設定をインポート - "external-services/example.com" # external-services 名前空間から 'example.com' のみインポート
  {{< /text >}}

### `exportTo` {#exportto}

Istio の `VirtualService`、`DestinationRule`、`ServiceEntry` には `spec.exportTo` フィールドがあります。
また、`Service` には `networking.istio.io/exportTo` アノテーションを使えます。

依存先をワークロード所有者が制御できる `Sidecar` とは異なり、
`exportTo` は逆方向で、サービス所有者が自身のサービスの可視性を制御します。

例：この設定は `details` サービスを自身の名前空間と `client` 名前空間のみに可視にします：

{{< text yaml >}}
apiVersion: v1
kind: Service
metadata:
name: details
annotations:
networking.istio.io/exportTo: ".,client"
spec: ...
{{< /text >}}

### `DiscoverySelectors` {#discoveryselectors}

前述の制御はワークロードやサービス所有者レベルですが、
[`DiscoverySelectors`](/ja/docs/reference/config/istio.mesh.v1alpha1/#MeshConfig) はメッシュ全体の設定可視性を制御します。
ディスカバリセレクターで、コントロールプレーンから見える名前空間の条件を指定できます。
一致しない名前空間はコントロールプレーンから完全に無視されます。

これはインストール時に `meshConfig` の一部として設定できます。例：

{{< text yaml >}}
meshConfig:
discoverySelectors: - matchLabels: # 任意の名前空間で `istio-discovery=enabled` を許可
istio-discovery: enabled - matchLabels: # "kube-system" を許可。Kubernetes はこのラベルを各名前空間に自動付与
kubernetes.io/metadata.name: kube-system
{{< /text >}}

{{< warning >}}
Istiod は常に Kubernetes のすべての名前空間を監視します。
ただし、ディスカバリセレクターは初期処理で選択されなかったオブジェクトを無視し、コストを最小化します。
{{</ warning >}}

## よくある質問 {#frequently-asked-questions}

### ある設定のコストを知るには？ {#how-can-i-understand-the-cost-of-a-certain-configuration}

設定スコープを絞ることで最大の効果を得るには、各オブジェクトのコストを知ることが役立ちます。
残念ながら、簡単な答えはありません。スケーラビリティは多くの要因に依存します。ただし、一般的なガイドラインはあります：

Istio では、設定**変更**のコストが高くなります。再計算が必要になるためです。
`Endpoints` の変更（通常は Pod のスケールイン/アウト）は大幅に最適化されていますが、
それ以外の多くの設定はコストが高いです。コントローラーがオブジェクトを頻繁に変更する場合（意図せず発生することも！）、
特に影響が大きくなります。どの設定が変更されているかを検出するツールもあります：

- Istiod は各変更をログに記録します。例：`Push debounce stable 1 for config Gateway/default/gateway: ..., full=true`。
  これは `default` 名前空間の `Gateway` オブジェクトが変更されたことを示します。
  `full=false` は `Endpoint` の更新など最適化された場合を示します。
  注意：`Service` や `Endpoints` の変更も `ServiceEntry` として表示されます。
- Istiod は `pilot_k8s_cfg_events` や `pilot_k8s_reg_events` で各変更のメトリクスを公開します。
- `kubectl get <resource> --watch -oyaml --show-managed-fields` で、
  1 つまたは複数のオブジェクトの変更内容や変更者を確認できます。

[ヘッドレスサービス（Headless Services）](https://kubernetes.io/ja/docs/concepts/services-networking/service/#headless-services)
（[HTTP](/ja/docs/ops/configuration/traffic-management/protocol-selection/#explicit-protocol-selection)として宣言されていない場合）は、
インスタンス数の増減に応じて変化します。
そのため、大規模なヘッドレスサービスはコストが高くなり、`exportTo` などで除外するのに適しています。

### 設定したスコープ外のサービスに接続した場合は？ {#what-happens-if-i-connect-to-a-service-outside-of-my-scope}

スコープ機構のいずれかで除外されたサービスに接続しようとすると、
データプレーンは対象の情報を持たないため、
[未マッチトラフィック](/ja/docs/ops/configuration/traffic-management/traffic-routing/#unmatched-traffic)として扱われます。

### ゲートウェイの場合は？ {#what-about-gateways}

[ゲートウェイ](/ja/docs/setup/additional-setup/gateway/)は `exportTo` や `DiscoverySelectors` に従いますが、
`Sidecar` オブジェクトはゲートウェイには影響しません。
ただし、Sidecar と異なり、ゲートウェイはデフォルトでクラスタ全体の設定を持ちません。
各設定は明示的にゲートウェイにアタッチされるため、この問題はほぼ回避されます。

ただし、[現時点では](https://github.com/istio/istio/issues/29131) データプレーン設定の一部
（Envoy 用語で「クラスタ」）は、明示的に参照されていなくてもクラスタ全体に送信されます。
