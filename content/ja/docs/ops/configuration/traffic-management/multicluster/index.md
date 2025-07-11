---
title: マルチクラスターのトラフィック管理
description: メッシュ内のクラスター間でトラフィックを分散する方法を設定します。
weight: 70
keywords: [traffic-management, multicluster]
owner: istio/wg-networking-maintainers
test: no
---

マルチクラスターメッシュでは、クラスターのトポロジに特化したトラフィックルールが必要になる場合があります。本ドキュメントでは、マルチクラスターメッシュでトラフィックを管理するいくつかの方法について説明します。
このガイドを読む前に、以下を確認してください：

1. [デプロイメントモデル](/ja/docs/ops/deployment/deployment-models/#multiple-clusters)を読んでください。
1. デプロイしたサービスが{{< gloss "namespace sameness" >}}ネームスペースの同一性{{< /gloss >}}の概念に従っていることを確認してください。

## クラスター内にトラフィックをとどめる {#keeping-traffic-in-cluster}

場合によっては、デフォルトのクラスタ間負荷分散動作が望ましくないことがあります。トラフィックを "cluster-local"（たとえば `cluster-a` から送信されたトラフィックが `cluster-a` 内の宛先のみに到達するように）に保つには、[`MeshConfig.serviceSettings`](/ja/docs/reference/config/istio.mesh.v1alpha1/#MeshConfig-ServiceSettings-Settings) でホスト名やワイルドカードを `clusterLocal` としてマークします。

たとえば、単一サービス、特定のネームスペース内のすべてのサービス、またはメッシュ内のすべてのサービスに対してグローバルに "cluster-local" トラフィック管理を適用できます：

{{< tabset category-name="meshconfig" >}}

{{< tab name="サービス単位" category-value="service" >}}

{{< text yaml >}}
serviceSettings:

- settings:
  clusterLocal: true
  hosts:
  - "mysvc.myns.svc.cluster.local"
    {{< /text >}}

{{< /tab >}}

{{< tab name="ネームスペース単位" category-value="namespace" >}}

{{< text yaml >}}
serviceSettings:

- settings:
  clusterLocal: true
  hosts:
  - "\*.myns.svc.cluster.local"
    {{< /text >}}

{{< /tab >}}

{{< tab name="グローバル" category-value="global" >}}

{{< text yaml >}}
serviceSettings:

- settings:
  clusterLocal: true
  hosts:
  - "\*"
    {{< /text >}}

{{< /tab >}}

{{< /tabset >}}

また、グローバルなクラスターローカルルールを設定し、明示的な例外（特定またはワイルドカード）を追加してサービスアクセスを細かく制御することもできます。
以下の例では、すべてのサービスがクラスターローカルになりますが、`myns` ネームスペース内のサービスは除外されます：

{{< text yaml >}}
serviceSettings:

- settings:
  clusterLocal: true
  hosts:
  - "\*"
- settings:
  clusterLocal: false
  hosts:
  - "\*.myns.svc.cluster.local"
    {{< /text >}}

## サービスの分割 {#partitioning-services}

[`DestinationRule.subsets`](/ja/docs/reference/config/networking/destination-rule/#Subset) を使うと、ラベルでサービスを分割できます。これらのラベルは Kubernetes metadata からのもの、または[組み込みラベル](/ja/docs/reference/config/labels/)のいずれかです。
組み込みラベルの 1 つである `topology.istio.io/cluster` を `DestinationRule` のサブセットセレクターで使うことで、クラスタごとにサブセットを作成できます。

{{< text yaml >}}
apiVersion: networking.istio.io/v1
kind: DestinationRule
metadata:
name: mysvc-per-cluster-dr
spec:
host: mysvc.myns.svc.cluster.local
subsets:

- name: cluster-1
  labels:
  topology.istio.io/cluster: cluster-1
- name: cluster-2
  labels:
  topology.istio.io/cluster: cluster-2
  {{< /text >}}

これらのサブセットを使って、[ミラーリング](/ja/docs/tasks/traffic-management/mirroring/)や[トラフィックシフト](/ja/docs/tasks/traffic-management/traffic-shifting/)など、さまざまなルーティングルールをクラスタごとに作成できます。

このサブセットベースのルーティング方式を使うことで、クラスタ内トラフィックを制御できますが、[`MeshConfig.serviceSettings`](/ja/docs/reference/config/istio.mesh.v1alpha1/#MeshConfig-ServiceSettings-Settings) には欠点もあります。サービスレベルのプロキシとトポロジレベルのプロキシが混在するためです。
たとえば、あるルールで 10% のトラフィックを `v2` サービスに送る場合、サブセットの数が 2 倍（例：`cluster-1-v2`、`cluster-2-v2`）必要になります。
この方法は、クラスタールーティングをより細かく制御したい場合に限定して使うのがよいでしょう。
