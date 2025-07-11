---
title: Helm を使ったアップグレード
description: Helm を使って Ambient モードのインストールをアップグレードする。
weight: 5
aliases:
  - /ja/docs/ops/ambient/upgrade/helm-upgrade
  - /ja/latest/docs/ops/ambient/upgrade/helm-upgrade
  - /ja/docs/ambient/upgrade/helm
  - /ja/latest/docs/ambient/upgrade/helm
owner: istio/wg-environments-maintainers
test: yes
status: Experimental
---

このガイドでは [Helm](https://helm.sh/docs/) を使って Ambient モードのインストールをアップグレードおよび設定する方法を説明します。このガイドは、以前のバージョンの Istio で[Helm を使った Ambient モードのインストール](/ja/docs/ambient/install/helm/)を実施済みであることを前提としています。

{{< warning >}}
Sidecar モードと比較して、Ambient モードではアプリケーション Pod をアップグレード後の ztunnel プロキシに移動する際、実行中のアプリケーション Pod を強制的に再起動したり再スケジュールしたりする必要がありません。しかし、**リビジョンを使っても**、ztunnel のアップグレードは**必ず**アップグレード対象ノード上のすべての長寿命 TCP 接続をリセットします。また、Istio は現時点で ztunnel のカナリアアップグレードをサポートしていません。

本番環境でのアップグレード時の影響範囲リスクを軽減するため、ノードドレインやブルー/グリーンノードプールの利用を推奨します。詳細は Kubernetes プロバイダーのドキュメントを参照してください。
{{< /warning >}}

## Ambient モードのアップグレードについて理解する {#understanding-ambient-mode-upgrades}

すべての Istio アップグレードは、コントロールプレーン、データプレーン、および Istio CRD のアップグレードを含みます。Ambient データプレーンは[2 つのコンポーネント](/ja/docs/ambient/architecture/data-plane)（ztunnel とゲートウェイ（waypoint を含む））に分かれているため、これらのコンポーネントごとに個別のアップグレード手順が必要です。ここではコントロールプレーンと CRD のアップグレードについて簡単に説明しますが、基本的には[Sidecar モードでのアップグレード手順](/ja/docs/setup/upgrade/canary/)と同じです。

Sidecar モードと同様に、ゲートウェイは[リビジョンラベル](/ja/docs/setup/upgrade/canary/#stable-revision-labels)を使って
{{< gloss >}}Gateway{{</ gloss >}}（waypoint を含む）のアップグレードを細かく制御でき、コントロールプレーンの旧バージョンへのロールバックも簡単に行えます。しかし、Sidecar モードと異なり、ztunnel は DaemonSet（ノードごとのプロキシ）として動作するため、ztunnel のアップグレードは少なくとも 1 度はノード全体に影響します。多くの場合これは許容範囲ですが、長時間 TCP 接続を持つアプリケーションでは中断が発生する可能性があります。その場合は、該当ノードの ztunnel をアップグレードする前にノードドレインや排出を推奨します。簡単のため、このドキュメントでは ztunnel のインプレースアップグレード（短時間のダウンタイムを伴う）を例示します。

## 前提条件 {#prerequisites}

### アップグレードの準備 {#prepare-for-the-upgrade}

Istio をアップグレードする前に、新しいバージョンの istioctl をダウンロードし、`istioctl x precheck` を実行してアップグレードが Ambient と互換性があることを確認することを推奨します。出力例は以下の通りです：

{{< text syntax=bash snip_id=istioctl_precheck >}}
$ istioctl x precheck
✔ No issues found when checking the cluster. Istio is safe to install or upgrade!
To get started, check out <https://istio.io/latest/docs/setup/getting-started/>
{{< /text >}}

次に、Helm リポジトリを更新します：

{{< text syntax=bash snip_id=update_helm >}}
$ helm repo update istio
{{< /text >}}

{{< tabset category-name="upgrade-prerequisites" >}}

{{< tab name="インプレースアップグレード" category-value="in-place" >}}

インプレースアップグレードには追加の準備は不要です。次のステップに進んでください。

{{< /tab >}}

{{< tab name="リビジョンアップグレード" category-value="revisions" >}}

### タグとリビジョンの整理 {#organize-your-tags-and-revisions}

Ambient モードのメッシュを制御しやすくアップグレードするため、ゲートウェイやネームスペースに `istio.io/rev` ラベルを付与し、どのゲートウェイやコントロールプレーンバージョンがワークロードのトラフィックを管理するかを制御することを推奨します。プロダクション環境では複数のラベルでアップグレードを段階的に行うのが理想です。ラベルごとに同時にアップグレードされるため、リスクの低いアプリケーションから始めるのが良いでしょう。ラベルで直接リビジョンを参照してアップグレードするのは、意図せず多くのプロキシを一度にアップグレードしてしまうリスクがあるため推奨しません。クラスタ内で使用中のラベルやリビジョンは、アップグレードラベルのセクションを参照してください。

### リビジョン名の選択 {#choose-a-revision-name}

リビジョンは Istio コントロールプレーンの一意のインスタンスを識別し、1 つのメッシュ内で複数バージョンのコントロールプレーンを同時に動作させることができます。

リビジョン名は一度決めたら変更せず、再利用もしないことを推奨します。一方、ラベルはリビジョンへの可変ポインタとして機能し、データプレーンのアップグレード時にワークロードのラベルを変更せずにラベルの参照先を新しいリビジョンに切り替えるだけで済みます。すべてのデータプレーンは `istio.io/rev` ラベル（またはデフォルトリビジョン）で指定されたコントロールプレーンにのみ接続します。データプレーンのアップグレードは、ラベルの参照先を変更するだけで完了します。

リビジョンは不変であるため、インストールする Istio バージョンに対応したリビジョン名（例：`1-22-1`）を推奨します。新しいリビジョン名を決めたら、現在のリビジョン名も控えておきましょう。以下のコマンドで確認できます：

{{< text syntax=bash snip_id=list_revisions >}}
$ kubectl get mutatingwebhookconfigurations -l 'istio.io/rev,!istio.io/tag' -L istio\.io/rev
$ # 現在のリビジョンと新しいリビジョンを変数に保存：
$ export REVISION=istio-1-22-1
$ export OLD_REVISION=istio-1-21-2
{{< /text >}}

{{< /tab >}}

{{< /tabset >}}

## コントロールプレーンのアップグレード {#upgrade-the-control-plane}

### 基本コンポーネント {#base-components}

{{< boilerplate crd-upgrade-123 >}}

新しいコントロールプレーンをデプロイする前に、クラスタ全体の Custom Resource Definitions（CRD）をアップグレードする必要があります：

{{< text syntax=bash snip_id=upgrade_crds >}}
$ helm upgrade istio-base istio/base -n istio-system
{{< /text >}}

### istiod コントロールプレーン {#istiod-control-plane}

[Istiod](/ja/docs/ops/deployment/architecture/#istiod) コントロールプレーンは、メッシュ内のトラフィックをルーティングするプロキシの管理と設定を行います。以下のコマンドは、現在のインスタンスの隣に新しいコントロールプレーンインスタンスをインストールしますが、新しいゲートウェイプロキシや waypoint を導入したり、既存プロキシの制御権を奪ったりはしません。

istiod のインストールをカスタマイズしている場合は、以前のアップグレードやインストールで使用した `values.yaml` ファイルを再利用してコントロールプレーンの一貫性を保つことができます。

{{< tabset category-name="upgrade-control-plane" >}}

{{< tab name="インプレースアップグレード" category-value="in-place" >}}

{{< text syntax=bash snip_id=upgrade_istiod_inplace >}}
$ helm upgrade istiod istio/istiod -n istio-system --wait
{{< /text >}}

{{< /tab >}}

{{< tab name="リビジョンアップグレード" category-value="revisions" >}}

{{< text syntax=bash snip_id=upgrade_istiod_revisioned >}}
$ helm install istiod-"$REVISION" istio/istiod -n istio-system --set revision="$REVISION" --set profile=ambient --wait
{{< /text >}}

{{< /tab >}}

{{< /tabset >}}

### CNI ノードエージェント {#cni-node-agent}

Istio CNI ノードエージェントは、Ambient メッシュに追加された Pod を検出し、ztunnel に追加された Pod 内でプロキシポートを確立するよう通知し、Pod のネットワーク名前空間内でトラフィックリダイレクトを設定します。これはデータプレーンやコントロールプレーンの一部ではありません。

1.x バージョンの CNI は 1.x+1 および 1.x バージョンのコントロールプレーンと互換性があります。つまり、コントロールプレーンと Istio CNI のバージョン差が 1 マイナーバージョン以内であれば、コントロールプレーンをアップグレードする前に CNI をアップグレードする必要があります。

{{< warning >}}
**リビジョンを使っても**、Istio は現時点で istio-cni のカナリアアップグレードをサポートしていません。Ambient で重大な影響がある場合や、CNI アップグレードの影響範囲を厳密に制御したい場合は、ノード自体の排出とアップグレードまで `istio-cni` のアップグレードを遅らせるか、ノードのテイントを利用して手動でこのコンポーネントのアップグレードを調整することを推奨します。

Istio CNI ノードエージェントは[システムノードクリティカル](https://kubernetes.io/ja/docs/tasks/administer-cluster/guaranteed-scheduling-critical-addon-pods/)な DaemonSet です。各ノードで稼働していなければ、そのノード上の Istio Ambient トラフィックのセキュリティと運用保証が維持できません。デフォルトで、Istio CNI ノードエージェント DaemonSet は安全なインプレースアップグレードをサポートし、アップグレードや再起動時には新しい Pod の起動をブロックします（そのノードに利用可能なエージェントが存在するまで）。これにより安全でないトラフィックの漏洩を防ぎます。アップグレード前に Ambient メッシュに正常に追加された既存 Pod は、アップグレード中も Istio のトラフィックセキュリティ要件に従って動作し続けます。
{{< /warning >}}

{{< text syntax=bash snip_id=upgrade_cni >}}
$ helm upgrade istio-cni istio/cni -n istio-system
{{< /text >}}

## データプレーンのアップグレード {#upgrade-the-data-plane}

### ztunnel DaemonSet {#ztunnel-daemonset}

{{< gloss >}}ztunnel{{< /gloss >}} DaemonSet はノードプロキシコンポーネントです。1.x バージョンの ztunnel は 1.x+1 および 1.x バージョンのコントロールプレーンと互換性があります。つまり、コントロールプレーンのバージョン差が 1 マイナーバージョン以内であれば、ztunnel をアップグレードする前にコントロールプレーンをアップグレードする必要があります。ztunnel のインストールをカスタマイズしている場合は、以前のアップグレードやインストールで使用した `values.yaml` ファイルを再利用して{{< gloss "data plane" >}}データプレーン{{< /gloss >}}の一貫性を保つことができます。

{{< warning >}}
**リビジョンを使っても**、インプレースアップグレードではノード上のすべての Ambient メッシュトラフィックが一時的に中断されます。実際には中断時間は非常に短く、主に長時間接続に影響します。

本番環境でのアップグレード時の影響範囲リスクを軽減するため、ノードドレインやブルー/グリーンノードプールの利用を推奨します。詳細は Kubernetes プロバイダーのドキュメントを参照してください。
{{< /warning >}}

{{< tabset category-name="upgrade-ztunnel" >}}

{{< tab name="インプレースアップグレード" category-value="in-place" >}}

{{< text syntax=bash snip_id=upgrade_ztunnel_inplace >}}
$ helm upgrade ztunnel istio/ztunnel -n istio-system --wait
{{< /text >}}

{{< /tab >}}

{{< tab name="リビジョンアップグレード" category-value="revisions" >}}

{{< text syntax=bash snip_id=upgrade_ztunnel_revisioned >}}
$ helm upgrade ztunnel istio/ztunnel -n istio-system --set revision="$REVISION" --wait
{{< /text >}}

{{< /tab >}}

{{< /tabset >}}

{{< tabset category-name="change-gateway-revision" >}}

{{< tab name="インプレースアップグレード" category-value="in-place" >}}

### 手動デプロイした Gateway チャートのアップグレード（オプション） {#upgrade-manually-deployed-gateway-chart-optional}

[手動デプロイ](/ja/docs/tasks/traffic-management/ingress/gateway-api/#manual-deployment)した `Gateway` は Helm で個別にアップグレードする必要があります：

{{< text syntax=bash snip_id=none >}}
$ helm upgrade istio-ingress istio/gateway -n istio-ingress
{{< /text >}}

{{< /tab >}}

{{< tab name="リビジョンアップグレード" category-value="revisions" >}}

### タグを使った waypoint とゲートウェイのアップグレード {#upgrade-waypoints-and-gateways-using-tags}

ベストプラクティスに従っている場合、すべてのゲートウェイ、ワークロード、ネームスペースはデフォルトリビジョン（実際には `default` という名前のタグ）または `istio.io/rev` ラベルを使い、その値をタグ名に設定しています。これらを新しいバージョンにアップグレードするには、タグの参照先を新バージョンに順次切り替えます。クラスタ内のすべてのタグを一覧表示するには、次のコマンドを実行します：

{{< text syntax=bash snip_id=list_tags >}}
$ kubectl get mutatingwebhookconfigurations -l 'istio.io/tag' -L istio\.io/tag,istio\.io/rev
{{< /text >}}

各タグについて、次のコマンドでタグをアップグレードできます。`$MYTAG` をタグ名、`$REVISION` をリビジョン名に置き換えてください：

{{< text syntax=bash snip_id=upgrade_tag >}}
$ helm template istiod istio/istiod -s templates/revision-tags.yaml --set revisionTags="{$MYTAG}" --set revision="$REVISION" -n istio-system | kubectl apply -f -
{{< /text >}}

これにより、そのタグを参照しているすべてのオブジェクトがアップグレードされます（[手動ゲートウェイデプロイ](/ja/docs/tasks/traffic-management/ingress/gateway-api/#manual-deployment)や Ambient モードでない Sidecar を除く）。

アップグレード後のデータプレーンを利用するアプリケーションの状態を十分に監視し、問題があればタグを旧リビジョンに戻してロールバックできます：

{{< text syntax=bash snip_id=rollback_tag >}}
$ helm template istiod istio/istiod -s templates/revision-tags.yaml --set revisionTags="{$MYTAG}" --set revision="$OLD_REVISION" -n istio-system | kubectl apply -f -
{{< /text >}}

### 手動デプロイしたゲートウェイのアップグレード（オプション） {#upgrade-manually-deployed-gateways-optional}

[手動デプロイ](/ja/docs/tasks/traffic-management/ingress/gateway-api/#manual-deployment)した `Gateway` は Helm で個別にアップグレードする必要があります：

{{< text syntax=bash snip_id=upgrade_gateway >}}
$ helm upgrade istio-ingress istio/gateway -n istio-ingress
{{< /text >}}

## 旧コントロールプレーンのアンインストール {#uninstall-the-previous-control-plane}

すべてのデータプレーンコンポーネントを新しいコントロールプレーンバージョンにアップグレードし、ロールバックが不要と判断した場合は、次のコマンドで旧コントロールプレーンを削除できます：

{{< text syntax=bash snip_id=delete_old_revision >}}
$ helm delete istiod-"$OLD_REVISION" -n istio-system
{{< /text >}}

{{< /tab >}}

{{< /tabset >}}
