---
title: Helm でのアップグレード
description: Helm を使った Istio アップグレード手順。
weight: 27
keywords: [kubernetes, helm]
owner: istio/wg-environments-maintainers
test: yes
---

このガイドでは [Helm](https://helm.sh/docs/) を使って Istio メッシュをアップグレード・設定する方法を説明します。
このガイドは、Istio の前のマイナーバージョンまたはパッチバージョンを[Helm でインストール](/ja/docs/setup/install/helm)済みであることを前提としています。

{{< boilerplate helm-preamble >}}

{{< boilerplate helm-prereqs >}}

## アップグレード手順 {#upgrade-steps}

Istio をアップグレードする前に、`istioctl x precheck` コマンドを実行して、アップグレードが環境と互換性があるか確認することを推奨します。

{{< text bash >}}
$ istioctl x precheck
✔ No issues found when checking the cluster. Istio is safe to install or upgrade!
To get started, check out <https://istio.io/latest/docs/setup/getting-started/>
{{< /text >}}

### カナリアアップグレード（推奨） {#canary-upgrade}

以下の手順でカナリアバージョンの Istio コントロールプレーンをインストールし、新バージョンが既存の設定やデータプレーンと互換性があるか検証できます：

{{< warning >}}
カナリアバージョンの `istiod` サービスをインストールする場合、Base Chart からのクラスタスコープリソースはメインインストールとカナリアインストールで共有されます。
{{< /warning >}}

{{< boilerplate crd-upgrade-123 >}}

1. Istio Base Chart をアップグレードし、すべてのクラスタスコープリソースを最新にします：

   {{< text bash >}}
   $ helm upgrade istio-base istio/base -n istio-system
   {{< /text >}}

1. revision 値を設定してカナリアバージョンの Istio Discovery Chart をインストールします：

   {{< text bash >}}
   $ helm install istiod-canary istio/istiod \
    --set revision=canary \
    -n istio-system
   {{< /text >}}

1. 2 つの `istiod` バージョンがクラスタにインストールされていることを確認します：

   {{< text bash >}}
   $ kubectl get pods -l app=istiod -L istio.io/rev -n istio-system
   NAME READY STATUS RESTARTS AGE REV
   istiod-5649c48ddc-dlkh8 1/1 Running 0 71m default
   istiod-canary-9cc9fd96f-jpc7n 1/1 Running 0 34m canary
   {{< /text >}}

1. [Istio Gateway](/ja/docs/setup/additional-setup/gateway/#deploying-a-gateway) を利用している場合、
   revision 値を設定してカナリアリビジョンの Gateway Chart をインストールします：

   {{< text bash >}}
   $ helm install istio-ingress-canary istio/gateway \
    --set revision=canary \
    -n istio-ingress
   {{< /text >}}

1. 2 つの `istio-ingress gateway` バージョンがクラスタにインストールされていることを確認します：

   {{< text bash >}}
   $ kubectl get pods -L istio.io/rev -n istio-ingress
   NAME READY STATUS RESTARTS AGE REV
   istio-ingress-754f55f7f6-6zg8n 1/1 Running 0 5m22s default
   istio-ingress-canary-5d649bd644-4m8lp 1/1 Running 0 3m24s canary
   {{< /text >}}

   [Gateway アップグレード](/ja/docs/setup/additional-setup/gateway/#canary-upgrade-advanced) も参照してください。

1. [こちら](/ja/docs/setup/upgrade/canary/#data-plane)の手順に従って、既存ワークロードをテストし、カナリアコントロールプレーンへ移行します。

1. カナリアコントロールプレーンでの動作を確認し、ワークロードを移行し終えたら、旧コントロールプレーンをアンインストールします：

   {{< text bash >}}
   $ helm delete istiod -n istio-system
   {{< /text >}}

1. Istio Base Chart を再度アップグレードし、新しい `canary` リビジョンをクラスタスコープのデフォルトバージョンに設定します：

   {{< text bash >}}
   $ helm upgrade istio-base istio/base --set defaultRevision=canary -n istio-system
   {{< /text >}}

### 安定リビジョンラベル（実験的機能） {#stable-revision-labels}

{{< boilerplate revision-tags-preamble >}}

#### 使い方 {#usage}

{{< boilerplate revision-tags-usage >}}

{{< text bash >}}
$ helm template istiod istio/istiod -s templates/revision-tags.yaml --set revisionTags="{prod-stable}" --set revision={{< istio_previous_version_revision >}}-1 -n istio-system | kubectl apply -f -
$ helm template istiod istio/istiod -s templates/revision-tags.yaml --set revisionTags="{prod-canary}" --set revision={{< istio_full_version_revision >}} -n istio-system | kubectl apply -f -
{{< /text >}}

{{< warning >}}
これらのコマンドは新しい `MutatingWebhookConfiguration` リソースをクラスタに作成しますが、
kubectl で手動適用するため、これらのリソースは Helm Chart に属しません。アンインストール方法は下記を参照してください。
{{< /warning >}}

{{< boilerplate revision-tags-middle >}}

{{< text bash >}}
$ helm template istiod istio/istiod -s templates/revision-tags.yaml --set revisionTags="{prod-stable}" --set revision={{< istio_full_version_revision >}} -n istio-system | kubectl apply -f -
{{< /text >}}

{{< boilerplate revision-tags-prologue >}}

#### デフォルトタグ {#default-tag}

{{< boilerplate revision-tags-default-intro >}}

{{< text bash >}}
$ helm template istiod istio/istiod -s templates/revision-tags.yaml --set revisionTags="{default}" --set revision={{< istio_full_version_revision >}} -n istio-system | kubectl apply -f -
{{< /text >}}

{{< boilerplate revision-tags-default-outro >}}

### インプレースアップグレード {#in-place-upgrade}

Helm アップグレードワークフローを使って、クラスタ内の Istio をインプレースアップグレードできます。

{{< warning >}}
Helm アップグレード時にカスタム設定を保持するには、上記コマンドにオーバーライド値ファイルやカスタムオプションを追加してください。
{{< /warning >}}

{{< boilerplate crd-upgrade-123 >}}

1. Istio Base Chart をアップグレードします：

   {{< text bash >}}
   $ helm upgrade istio-base istio/base -n istio-system
   {{< /text >}}

1. Istio Discovery Chart をアップグレードします：

   {{< text bash >}}
   $ helm upgrade istiod istio/istiod -n istio-system
   {{< /text >}}

1. （オプション）クラスタにインストールされている Gateway Chart もアップグレードします：

   {{< text bash >}}
   $ helm upgrade istio-ingress istio/gateway -n istio-ingress
   {{< /text >}}

## アンインストール {#uninstall}

[Helm インストールガイド](/ja/docs/setup/install/helm/#uninstall) のアンインストール手順を参照してください。
