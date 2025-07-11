---
title: Istioctl を使用したインストール
description: Istio 設定プロファイルをインストール、カスタマイズして、詳細な評価と本番デプロイメントを行います。
weight: 10
keywords: [istioctl, kubernetes]
owner: istio/wg-environments-maintainers
test: no
---

このガイドに従って、詳細な評価と本番デプロイメント用の Istio メッシュをインストールおよび設定してください。
Istio を初めて使用し、簡単に試してみたい場合は、[クイックスタートガイド](/ja/docs/setup/getting-started)を参照してください。

このインストールガイドでは、コマンドラインツール [istioctl](/ja/docs/reference/commands/istioctl/) を使用します。
これは、Istio コントロールプレーンとデータプレーン Sidecar をカスタマイズするための豊富なカスタマイズ機能を提供します。
また、ユーザー入力検証機能も提供し、インストールエラーの防止に役立ちます。さらに、設定のあらゆる側面をオーバーライドできるカスタマイズオプションを提供します。

これらの手順を使用して、Istio の組み込み[設定プロファイル](/ja/docs/setup/additional-setup/config-profiles/)のいずれかを選択し、
特定のニーズに合わせて設定をさらにカスタマイズできます。

`istioctl` コマンドは、コマンドラインオプションを通じて完全な
[`IstioOperator` API](/ja/docs/reference/config/istio.operator.v1alpha1/) をサポートし、
これらのオプションは個別設定や、IstioOperator {{<gloss CRD>}}カスタムリソース（CR）{{</gloss>}}を含む yaml ファイルの受信に使用されます。

## 前提条件 {#prerequisites}

開始する前に、以下の前提条件を確認してください：

1. [Istio リリースをダウンロード](/ja/docs/setup/additional-setup/download-istio-release/)します。
1. 必要な[プラットフォームセットアップ](/ja/docs/setup/platform-setup/)を実行します。
1. [Pod とサービスの要件](/ja/docs/ops/deployment/application-requirements/)を確認します。

## デフォルト設定プロファイルを使用した Istio のインストール {#install-using-default-profile}

最も簡単な選択肢は、以下のコマンドでデフォルトの[設定プロファイル](/ja/docs/setup/additional-setup/config-profiles/)を使用して Istio をインストールすることです：

{{< text bash >}}
$ istioctl install
{{< /text >}}

このコマンドは、Kubernetes クラスター上に `default` 設定プロファイルをインストールします。
`default` 設定プロファイルは本番環境を構築するための良い出発点であり、
評価で広範囲の Istio 機能によく使用される大きな `demo` 設定プロファイルとは異なります。

さまざまな設定を構成してインストールを変更できます。たとえば、アクセスログを有効にするには：

{{< text bash >}}
$ istioctl install --set meshConfig.accessLogFile=/dev/stdout
{{< /text >}}

{{< tip >}}
このページやドキュメントの他の場所にある多くの例では、設定ファイルを `-f` で渡すのではなく、
`--set` を使用してインストールパラメーターを変更しています。
これにより例がより簡潔になります。
この 2 つの方法は同等ですが、本番環境では `-f` の使用を強く推奨します。
上記のコマンドは `-f` を使用して以下のように書くことができます：

{{< text bash >}}
$ cat <<EOF > ./my-config.yaml
apiVersion: install.istio.io/v1alpha1
kind: IstioOperator
spec:
meshConfig:
accessLogFile: /dev/stdout
EOF
$ istioctl install -f my-config.yaml
{{< /text >}}

{{< /tip >}}

{{< tip >}}
完全な API は [`IstioOperator` API リファレンスドキュメント](/ja/docs/reference/config/istio.operator.v1alpha1/)に記載されています。
通常、Helm と同様に `istioctl` で `--set` パラメーターを使用でき、
現在の Helm の `values.yaml` API は下位互換性があります。
唯一の違いは、元の `values.yaml` パスの前に `values.` プレフィックスを付ける必要があることです。これは Helm パススルー API のプレフィックスです。
{{< /tip >}}

## 外部 Chart からのインストール {#install-from-external-charts}

デフォルトでは、`istioctl` は組み込み Chart を使用してインストールマニフェストを生成します。
これらの Chart は `istioctl` と一緒にリリースされ、監査とカスタマイズのために、リリースパッケージの `manifests` ディレクトリで見つけることができます。
`istioctl` は組み込み Chart を使用する以外に、外部 Chart も使用できます。
外部 Chart を選択するには、パラメーター `manifests` をローカルファイルシステムパスに設定します：

{{< text bash >}}
$ istioctl install --manifests=manifests/
{{< /text >}}

`istioctl` {{< istio_full_version >}} バージョンのバイナリを使用する場合、このコマンドは単独で `istioctl install` を実行するのと同じ結果を得ます。
これは、組み込み Chart と同じ Chart を指しているためです。
新機能の実験やテストを行う場合を除き、`istioctl` と Chart の互換性を保証するため、外部 Chart ではなく組み込み Chart の使用を推奨します。

## 異なる設定プロファイルのインストール {#install-a-different-profile}

他の Istio 設定プロファイルは、コマンドラインで設定プロファイル名を渡すことでクラスターにインストールできます。
たとえば、以下のコマンドで `demo` 設定プロファイルをインストールできます。

{{< text bash >}}
$ istioctl install --set profile=demo
{{< /text >}}

## インストール前のマニフェストファイル生成 {#generate-a-manifest-before-installation}

Istio をインストールする前に、`manifest generate`
サブコマンドを使用してマニフェストファイルを生成できます。たとえば、以下のコマンドを使用して
`kubectl` でインストールできる `default` 設定プロファイルのマニフェストを生成します：

{{< text bash >}}
$ istioctl manifest generate > $HOME/generated-manifest.yaml
{{< /text >}}

生成されたマニフェストは、具体的に何がインストールされたかを検査し、時間の経過とともにマニフェストの変更を追跡するために使用できます。
`IstioOperator` CR は完全なユーザー設定を表し、それを追跡するのに十分ですが、
`manifest generate` の出力は基盤となる Chart の可能な変更もキャプチャするため、
実際にインストールされたリソースを追跡するために使用できます。

{{< tip >}}
通常インストールに使用する他のフラグやカスタム値のオーバーライドも、`istioctl manifest generate` コマンドに提供する必要があります。
{{< /tip >}}

{{< warning >}}
`istioctl manifest generate` を使用して Istio をインストールおよび管理しようとする場合は、以下のことに注意してください：

1. Istio の名前空間（デフォルトは `istio-system`）を手動で作成する必要があります。

1. デフォルトでは、Istio 検証は有効になりません。
   `istioctl install` とは異なり、`manifest generate` コマンドは `values.defaultRevision` が設定されていない限り、`istiod-default-validator` 検証 webhook 設定を作成しません：

   {{< text bash >}}
   $ istioctl manifest generate --set values.defaultRevision=default
   {{< /text >}}

1. リソースは `istioctl install` と同じ依存関係の順序でインストールされない可能性があります。

1. この方法は、Istio リリースの一部としてテストされていません。

1. `istioctl install` は Kubernetes コンテキストで環境固有の設定を自動的に検出しますが、
   オフラインで実行される `manifest generate` はそれができず、予期しない結果を招く可能性があります。
   特に、Kubernetes 環境がサードパーティサービスアカウントトークンをサポートしていない場合は、
   [これらの手順](/ja/docs/ops/best-practices/security/#configure-third-party-service-account-tokens)に従うことを確認する必要があります。
   `istio manifest generate` コマンドの後に `--cluster-specific` を付けてターゲットクラスターの環境を検出することを推奨します。
   これにより、これらのクラスター固有の環境設定が生成されたマニフェストに埋め込まれます。これには実行中のクラスターへのネットワークアクセスが必要です。

1. `kubectl apply` で生成されたマニフェストを実行すると、一時的なエラーが表示されます。
   これは、クラスター内のリソースが利用可能な状態になる順序に問題があるためです。

1. `istioctl install` は、設定変更時（たとえば、ゲートウェイを削除した場合）に削除すべき一部のリソースを自動的にクリーンアップします。
   しかし、このメカニズムは `kubectl` と `istio manifest generate` を
   協調して使用する場合には発生しないため、これらのリソースは手動で削除する必要があります。

{{< /warning >}}

インストールのカスタマイズの詳細については、[インストール設定のカスタマイズ](/ja/docs/setup/additional-setup/customize-installation/)を参照してください。

## Istio のアンインストール {#uninstall}

クラスターから Istio を完全にアンインストールするには、以下のコマンドを実行します：

{{< text bash >}}
$ istioctl uninstall --purge
{{< /text >}}

{{< warning >}}
オプションの `--purge` パラメーターは、他の Istio コントロールプレーンによって共有される可能性のあるクラスター全体のリソースを含む、すべての Istio リソースを削除します。
{{< /warning >}}

または、指定された Istio コントロールプレーンのみを削除するには、以下のコマンドを実行します：

{{< text bash >}}
$ istioctl uninstall <元のインストールオプション>
{{< /text >}}

または

{{< text bash >}}
$ istioctl manifest generate <元のインストールオプション> | kubectl delete --ignore-not-found=true -f -
{{< /text >}}

コントロールプレーンの名前空間（例：`istio-system`）はデフォルトでは削除されません。
不要であることが確認できたら、以下のコマンドでその名前空間を削除します：

{{< text bash >}}
$ kubectl delete namespace istio-system
{{< /text >}}
