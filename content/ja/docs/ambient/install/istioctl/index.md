---
title: istioctl でのインストール
description: istioctl コマンドラインツールを使って Ambient モード対応の Istio をインストールします。
weight: 10
keywords: [istioctl, ambient]
owner: istio/wg-environments-maintainers
test: yes
---

{{< tip >}}
このガイドに従って、Ambient モード対応の Istio メッシュをインストールおよび構成します。
Istio を初めて使用する場合や、試してみたいだけの場合は、[クイックスタートガイド](/ja/docs/ambient/getting-started)に従ってください。
{{< /tip >}}

本インストールガイドでは [istioctl](/ja/docs/reference/commands/istioctl/)
コマンドラインツールを使用します。他のインストール方法と同様に、`istioctl`
は多くのカスタマイズオプションを提供します。また、ユーザー入力の検証機能によりインストールミスを防ぎ、
インストール後の分析や設定ツールも多数含まれています。

この手順では、Istio の組み込み[プロファイル](/ja/docs/setup/additional-setup/config-profiles/)のいずれかを選択し、
さらにご自身のニーズに合わせて設定をカスタマイズできます。

`istioctl` コマンドはコマンドラインオプションで個別に設定するか、
`IstioOperator` {{<gloss CRD>}}カスタムリソース{{</gloss>}} を含む YAML ファイルを渡して、
[`IstioOperator` API](/ja/docs/reference/config/istio.operator.v1alpha1/) の全機能を利用できます。

## 前提条件 {#prerequisites}

始める前に、以下の前提条件を確認してください：

1. [Istio リリースをダウンロード](/ja/docs/setup/additional-setup/download-istio-release/)してください。
1. 必要な[プラットフォーム固有の設定](/ja/docs/ambient/install/platform-prerequisites/)を実施してください。

## Kubernetes Gateway API CRD のインストールまたはアップグレード {#install-or-upgrade-the-kubernetes-gateway-api-crds}

{{< boilerplate gateway-api-install-crds >}}

## Ambient プロファイルで Istio をインストールする {#install-istio-using-the-ambient-profile}

`istioctl` は複数の[プロファイル](/ja/docs/setup/additional-setup/config-profiles/)をサポートしており、
それぞれ異なるデフォルトオプションが含まれており、本番環境のニーズに合わせてカスタマイズできます。
`ambient` プロファイルには Ambient モードのサポートが含まれています。次のコマンドで Istio をインストールします：

{{< text syntax=bash snip_id=install_ambient >}}
$ istioctl install --set profile=ambient --skip-confirmation
{{< /text >}}

このコマンドは、Kubernetes 設定で定義されたクラスターに `ambient` プロファイルをインストールします。

## プロファイルの設定とカスタマイズ {#configure-and-modify-profiles}

Istio のインストール API は
[`IstioOperator` API リファレンス](/ja/docs/reference/config/istio.operator.v1alpha1/)に記載されています。
`istioctl install` の `--set` オプションで各インストールパラメータを変更したり、`-f` で独自の設定ファイルを指定できます。

`istioctl` インストールの使い方やカスタマイズの詳細は、[サイドカーインストールドキュメント](/ja/docs/setup/install/istioctl/)を参照してください。

## Istio のアンインストール {#uninstall-istio}

Istio をクラスターから完全にアンインストールするには、次のコマンドを実行します：

{{< text syntax=bash snip_id=uninstall >}}
$ istioctl uninstall --purge -y
{{< /text >}}

{{< warning >}}
オプションの `--purge` フラグを付けると、すべての Istio リソースが削除され、
他の Istio コントロールプレーンと共有されているクラスタ範囲のリソースも削除されます。
{{< /warning >}}

または、特定の Istio コントロールプレーンのみを削除したい場合は、次のコマンドを実行します：

{{< text syntax=bash snip_id=none >}}
$ istioctl uninstall <your original installation options>
{{< /text >}}

コントロールプレーンの名前空間（例：`istio-system`）はデフォルトでは削除されません。
不要な場合は、次のコマンドで削除できます：

{{< text syntax=bash snip_id=remove_namespace >}}
$ kubectl delete namespace istio-system
{{< /text >}}

## インストール前のマニフェスト生成 {#generate-a-manifest-before-installation}

Istio をインストールする前に、`manifest generate` サブコマンドでマニフェストを生成できます。
たとえば、`default` プロファイル用のマニフェストを生成し、`kubectl` でインストールするには：

{{< text syntax=bash snip_id=none >}}
$ istioctl manifest generate > $HOME/generated-manifest.yaml
{{< /text >}}

生成されたマニフェストは、何がインストールされるかの確認や、マニフェストの変更履歴の追跡に利用できます。
`IstioOperator` CR はユーザー設定全体を表し、追跡には十分ですが、
`manifest generate` の出力は基盤となるチャートの変更も反映するため、実際にインストールされるリソースの追跡にも役立ちます。

{{< tip >}}
インストール時に使う他のフラグやカスタム値の上書きも、`istioctl manifest generate` コマンドに渡す必要があります。
{{< /tip >}}

{{< warning >}}
`istioctl manifest generate` で Istio をインストール・管理する場合、以下の注意点があります：

1. Istio の名前空間（デフォルトは `istio-system`）は手動で作成する必要があります。

1. Istio のバリデーションはデフォルトで有効になりません。`istioctl install` とは異なり、
   `manifest generate` コマンドは `istiod-default-validator` バリデーション Webhook 設定を作成しません。
   これを有効にするには `values.defaultRevision` を設定してください：

   {{< text syntax=bash snip_id=none >}}
   $ istioctl manifest generate --set values.defaultRevision=default
   {{< /text >}}

1. リソースは `istioctl install` と同じ依存順でインストールされない場合があります。

1. この方法は Istio のリリースでテストされていません。

1. `istioctl install` は Kubernetes コンテキストから環境固有の設定を自動検出しますが、
   `manifest generate` はオフラインで動作するためそれができず、予期しない結果になる場合があります。特に、Kubernetes 環境がサードパーティサービスアカウントトークンをサポートしていない場合、[これらの手順](/ja/docs/ops/best-practices/security/#configure-third-party-service-account-tokens)に従う必要があります。
   `istio manifest generate` コマンドに `--cluster-specific` を付けることで、ターゲットクラスタの環境設定を検出し、生成されるマニフェストに埋め込むことができます。
   これには稼働中のクラスタへのネットワークアクセスが必要です。

1. クラスタ内のリソースが正しい順序で利用できない場合、生成されたマニフェストの `kubectl apply` で一時的なエラーが発生することがあります。

1. `istioctl install` は設定変更時に削除すべきリソース（例：ゲートウェイの削除）を自動で削除しますが、
   `istio manifest generate` と `kubectl` の組み合わせでは自動削除されないため、手動で削除する必要があります。

{{< /warning >}}
