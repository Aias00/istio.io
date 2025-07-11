---
title: ワークロードをメッシュに追加する
description: Ambient メッシュにワークロードを追加する方法を学びます。
weight: 10
owner: istio/wg-networking-maintainers
test: no
---

ほとんどの場合、クラスター管理者が Istio メッシュのインフラストラクチャをデプロイします。
Ambient {{< gloss "data plane" >}}データプレーン{{< /gloss >}}モードをサポートする Istio が正常にデプロイされると、Istio を利用できるように設定されたネームスペース内のすべてのユーザーがデプロイしたアプリケーションで Istio を完全に利用できます。

## メッシュ内のアプリケーションで Ambient モードを有効にする {#enabling-ambient-mode-for-an-application-in-the-mesh}

Ambient モードでアプリケーションやネームスペースをメッシュに追加するには、該当リソースに `istio.io/dataplane-mode=ambient` ラベルを追加します。このラベルはネームスペースまたは個々の Pod に適用できます。

アプリケーション Pod については、完全に透過的かつシームレスに Ambient モードを有効化（または無効化）できます。
{{< gloss "sidecar" >}}Sidecar{{< /gloss >}} データプレーンモードとは異なり、アプリケーションをメッシュに追加する際に再起動は不要で、Pod 内に追加コンテナがデプロイされているようには見えません。

### Layer 4 と Layer 7 の機能 {#layer-4-and-layer-7-functionality}

セキュアな L4 カバレッジは認証と認可ポリシーをサポートします。
[Ambient モードでの L4 ポリシーサポートについてはこちら](/ja/docs/ambient/usage/l4-policy/)。
Istio の L7 機能（トラフィックルーティングなど）を利用するには、[waypoint プロキシをデプロイし、ワークロードを登録](/ja/docs/ambient/usage/waypoint/)する必要があります。

### Ambient と Kubernetes NetworkPolicy {#ambient-and-kubernetes-networkpolicy}

[Ambient と Kubernetes NetworkPolicy](/ja/docs/ambient/usage/networkpolicy/) を参照してください。

## 異なるデータプレーンモード間の Pod 通信 {#communicating-between-pods-in-different-data-plane-modes}

Ambient データプレーンモードのアプリケーション Pod と非 Ambient エンドポイント（Kubernetes アプリケーション Pod、Istio ゲートウェイ、Kubernetes Gateway API インスタンスなど）間の相互運用性には複数の選択肢があります。この相互運用性により、同じ Istio メッシュ内で Ambient と非 Ambient ワークロードをシームレスに統合でき、メッシュの導入や運用ニーズに合わせて段階的に Ambient 機能を導入できます。

### メッシュ外の Pod {#pods-outside-the-mesh}

Sidecar モードでも Ambient モードでも、ネームスペースがメッシュの一部でない場合があります。
この場合、非メッシュ Pod は直接ターゲット Pod へトラフィックを送信し、ソースノードの ztunnel を経由しませんが、ターゲット Pod の ztunnel は任意の L4 ポリシーを強制し、トラフィックの許可・拒否を制御します。

例えば、Ambient モードが有効なネームスペースで `PeerAuthentication` ポリシーの mTLS モードを `STRICT` に設定すると、メッシュ外からのトラフィックは拒否されます。

### Sidecar モードを利用するメッシュ内 Pod {#pods-inside-the-mesh-using-sidecar-mode}

Istio は同じメッシュ内で Sidecar を持つ Pod と Ambient モードの Pod の間のイースト・ウエスト相互運用性をサポートします。
ターゲットが HBONE ターゲットであることが検出されるため、Sidecar プロキシは HBONE プロトコルを使用することを認識します。

{{< tip >}}
Sidecar プロキシが Ambient ターゲットと通信する際に HBONE/mTLS シグナリングオプションを使用するには、プロキシメタデータで `ISTIO_META_ENABLE_HBONE` 設定を `true` にする必要があります。これは `ambient` プロファイルを使用する場合、`MeshConfig` でデフォルト設定されているため、追加の操作は不要です。
{{< /tip >}}

`PeerAuthentication` ポリシーの mTLS モードを `STRICT` に設定すると、Istio Sidecar プロキシを持つ Pod からのトラフィックが許可されます。

### イングレス・イーグレスゲートウェイと Ambient モード Pod {#ingress-and-egress-gateways-and-ambient-mode-pods}

イングレスゲートウェイは非 Ambient ネームスペースで動作し、Ambient モード、Sidecar モード、非メッシュ Pod が提供するサービスを公開できます。
Ambient モードの Pod と Istio イーグレスゲートウェイ間の相互運用性もサポートされています。

## Ambient モードと Sidecar モードの Pod 選択ロジック {#pod-selection-logic-for-ambient-and-sidecar-modes}

Istio の 2 つのデータプレーンモード（Sidecar と Ambient）は同じクラスター内で共存できます。
同じ Pod やネームスペースが両方のモードを同時に使用しないようにすることが重要です。
ただし、もし両方の設定がなされた場合、現時点ではその Pod やネームスペースは Sidecar モードが優先されます。

理論上、ネームスペースラベルとは別に個々の Pod にラベルを付与することで、同じネームスペース内の 2 つの Pod を異なるモードで動作させることも可能ですが、推奨されません。ほとんどの一般的なユースケースでは、単一モードを 1 つのネームスペース内のすべての Pod に適用します。

Pod が Ambient モードで動作するかどうかの判定ロジックは以下の通りです：

1. `cni.values.excludeNamespaces` で設定された `istio-cni` プラグインの除外リストに含まれるネームスペースはスキップされます。
1. Pod は以下の場合に `ambient` モードを使用します：
   - ネームスペースまたは Pod に `istio.io/dataplane-mode=ambient` ラベルがある
   - Pod に `istio.io/dataplane-mode=none` ラベルがない
   - Pod に `sidecar.istio.io/status` アノテーションがない

設定の競合を避ける最も簡単な方法は、各ネームスペースに対して Sidecar インジェクションラベル（`istio-injection=enabled`）または Ambient モードラベル（`istio.io/dataplane-mode=ambient`）のいずれか一方のみを付与し、両方を同時に付与しないことです。

## ラベルリファレンス {#ambient-labels}

以下のラベルは、リソースが Ambient モードのメッシュに含まれるかどうか、waypoint プロキシで L7 ポリシーを強制するかどうか、waypoint へのトラフィックの送信方法を制御します。

| 名称                      | 機能ステータス | リソース                               | 説明                                                                                                                                                                                     |
| ------------------------- | -------------- | -------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `istio.io/dataplane-mode` | Beta           | `Namespace` または `Pod`（Pod が優先） | リソースを Ambient メッシュに追加します。<br><br>有効値：`ambient` または `none`。                                                                                                       |
| `istio.io/use-waypoint`   | Beta           | `Namespace`、`Service` または `Pod`    | 対象リソースの waypoint を使って L7 ポリシーを適用します。<br><br>有効値：`{waypoint-name}` または `none`。                                                                              |
| `istio.io/waypoint-for`   | Alpha          | `Gateway`                              | waypoint が処理するトラフィックのエンドポイントタイプを指定します。<br><br>有効値：`service`、`workload`、`none` または `all`。このラベルはオプションで、デフォルト値は `service` です。 |

`istio.io/use-waypoint` ラベル値を有効にするには、トラフィックを処理するリソースタイプに対して waypoint を設定しておく必要があります。デフォルトでは、waypoint はサービストラフィックを受け入れます。たとえば、`istio.io/use-waypoint` ラベルで Pod を特定の waypoint に関連付ける場合、その waypoint には `istio.io/waypoint-for` ラベルが `workload` または `all` に設定されている必要があります。
