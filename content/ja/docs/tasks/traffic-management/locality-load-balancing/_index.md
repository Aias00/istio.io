---
title: ローカリティ負荷分散
description: このシリーズのタスクでは、Istio でローカリティ負荷分散を構成する方法を説明します。
weight: 65
keywords:
  [locality, load balancing, priority, prioritized, kubernetes, multicluster]
list_below: true
simple_list: true
content_above: true
aliases:
  - /help/ops/traffic-management/locality-load-balancing
  - /help/ops/locality-load-balancing
  - /help/tasks/traffic-management/locality-load-balancing
  - /zh/docs/ops/traffic-management/locality-load-balancing
  - /zh/docs/ops/configuration/traffic-management/locality-load-balancing
owner: istio/wg-networking-maintainers
test: n/a
---

**ローカリティ** とは、{{< gloss >}}ワークロードインスタンス{{</ gloss >}} がメッシュ内で存在する地理的な場所を定義します。ローカリティは次の 3 つの要素で構成されます：

- **リージョン**：大きな地理的領域を表します（例：**us-east**）。1 つのリージョンには複数の**ゾーン**が含まれます。
  Kubernetes では、ラベル [`topology.kubernetes.io/region`](https://kubernetes.io/ja/docs/reference/labels-annotations-taints/#topologykubernetesioregion) がノードのリージョンを決定します。

- **ゾーン**：リージョン内の計算リソースのグループです。複数のゾーンにサービスをデプロイすることで、ゾーン間でフェイルオーバーしつつ、ユーザーデータのローカリティを維持できます。
  Kubernetes では、ラベル [`topology.kubernetes.io/zone`](https://kubernetes.io/ja/docs/reference/labels-annotations-taints/#topologykubernetesiozone) がノードのゾーンを決定します。

- **サブゾーン**：管理者がゾーンをさらに細分化して、より細かい制御（例：「同一ラック」）を実現できます。
  Kubernetes にはサブゾーンの概念はありません。そのため、Istio ではカスタムノードラベル [`topology.istio.io/subzone`](/ja/docs/reference/config/labels/#:~:text=topology.istio.io/subzone) を導入しています。

{{< tip >}}
マネージド Kubernetes サービスを利用している場合、クラウドプロバイダーがリージョンとゾーンのラベルを設定してくれます。
独自の Kubernetes クラスタを運用している場合は、これらのラベルをノードに追加する必要があります。
{{< /tip >}}

ローカリティは階層構造で、次の順序でマッチします：

1. リージョン
1. ゾーン
1. サブゾーン

つまり、`foo` リージョンの `bar` ゾーンで動作する Pod は、`baz` リージョンの `bar` ゾーンで動作する Pod とは**みなされません**。

Istio はローカリティ情報を使って負荷分散の挙動を制御します。本シリーズのいずれかのタスクを参照して、メッシュにローカリティ負荷分散を構成してください。
