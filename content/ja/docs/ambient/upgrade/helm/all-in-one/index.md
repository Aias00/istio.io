---
title: Helm を使ったアップグレード（簡易）
description: 単一のチャートで Helm を使って Ambient モードのインストールをアップグレードする
weight: 5
owner: istio/wg-environments-maintainers
test: yes
draft: true
---

このガイドでは [Helm](https://helm.sh/docs/) を使って Ambient モードのインストールをアップグレードおよび設定する方法を説明します。このガイドは、以前のバージョンの Istio で[Helm と Ambient ラッパーチャートを使った Ambient モードのインストール](/ja/docs/ambient/install/helm/all-in-one)を実施済みであることを前提としています。

{{< warning >}}
これらのアップグレード手順は、Ambient ラッパーチャートで作成された Helm インストールのアップグレードにのみ適用されます。個別の Helm コンポーネントチャートでインストールした場合は、[関連するアップグレードドキュメント](/ja/docs/ambient/upgrade/helm)を参照してください。
{{< /warning >}}

## Ambient モードのアップグレードについて理解する {#understanding-ambient-mode-upgrades}

{{< warning >}}
すべてをこのラッパーチャートの一部としてインストールした場合、Ambient のアップグレードやアンインストールはこのラッパーチャート経由でのみ可能です。サブコンポーネントを個別にアップグレードやアンインストールすることはできません。
{{< /warning >}}

## 前提条件 {#prerequisites}

### アップグレードの準備 {#prepare-for-the-upgrade}

Istio をアップグレードする前に、新しいバージョンの istioctl をダウンロードし、`istioctl x precheck` を実行してアップグレードが環境と互換性があることを確認することを推奨します。出力例は以下の通りです：

{{< text syntax=bash snip_id=istioctl_precheck >}}
$ istioctl x precheck
✔ No issues found when checking the cluster. Istio is safe to install or upgrade!
To get started, check out <https://istio.io/latest/docs/setup/getting-started/>
{{< /text >}}

次に、Helm リポジトリを更新します：

{{< text syntax=bash snip_id=update_helm >}}
$ helm repo update istio
{{< /text >}}

### Istio Ambient コントロールプレーンとデータプレーンのアップグレード {#upgrade-the-istio-ambient-control-plane-and-data-plane}

{{< warning >}}
ラッパーチャートを使ったインプレースアップグレードでは、ノード上のすべての Ambient メッシュトラフィックが一時的に中断されます。
**リビジョンを使っても同様です**。実際には中断期間は非常に短く、主に長時間接続に影響します。

本番環境でのアップグレード時の影響範囲リスクを軽減するため、ノードドレインやブルー/グリーンノードプールの利用を推奨します。詳細は Kubernetes プロバイダーのドキュメントを参照してください。
{{< /warning >}}

`ambient` チャートは、各コンポーネントチャートで構成される Helm ラッパーチャートを使って、Ambient に必要なすべての Istio データプレーンおよびコントロールプレーンコンポーネントをアップグレードします。

istiod のインストールをカスタマイズしている場合は、以前のアップグレードやインストールで使用した `values.yaml` ファイルを再利用して設定を一貫させることができます。

{{< text syntax=bash snip_id=upgrade_ambient_aio >}}
$ helm upgrade istio-ambient istio/ambient -n istio-system --wait
{{< /text >}}

### 手動デプロイした Gateway チャートのアップグレード（オプション） {#upgrade-manually-deployed-gateway-chart-optional}

[手動デプロイ](/ja/docs/tasks/traffic-management/ingress/gateway-api/#manual-deployment)した `Gateway` は Helm で個別にアップグレードする必要があります：

{{< text syntax=bash snip_id=none >}}
$ helm upgrade istio-ingress istio/gateway -n istio-ingress
{{< /text >}}
