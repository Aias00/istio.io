---
title: MeshConfig と Pod アノテーションによるトレース設定
description: MeshConfig と Pod アノテーションを使ってトレースを設定する方法。
weight: 3
keywords: [telemetry, tracing]
aliases:
  - /zh/docs/tasks/observability/distributed-tracing/configurability/
  - /zh/docs/tasks/observability/distributed-tracing/configurability/mesh-and-proxy-config/
owner: istio/wg-policies-and-telemetry-maintainers
test: no
status: Beta
---

{{< boilerplate telemetry-tracing-tips >}}

Istio は、サンプリングレートやレポートされる span にカスタムタグを追加するなど、高度なトレースオプションの設定機能を提供します。

## 始める前に {#before-you-begin}

1. アプリケーションが[こちら](/zh/docs/tasks/observability/distributed-tracing/overview/)で説明されているようにトレースヘッダーを伝播していることを確認してください。

1. [統合](/zh/docs/ops/integrations/)の下にあるトレースインストールガイドに従い、希望するトレースバックエンドをインストールし、Istio プロキシがトレースをトレースデプロイメントに送信するように設定してください。

## 利用可能なトレース設定 {#available-tracing-configurations}

Istio では、以下のトレースオプションを設定できます：

1. トレースデータを生成するリクエストを一定の割合でランダムサンプリングします。

1. リクエストパスの最大長。これを超えるとパスは切り捨ててレポートされます。イングレスゲートウェイでトレースを収集する場合、トレースデータストレージの制限に役立ちます。

1. span にカスタムタグを追加できます。これらのタグは、静的な文字列、リクエストヘッダーの値、環境変数、またはフィールドに基づいて追加できます。
   これにより、span に環境固有の追加情報を注入できます。

トレースオプションの設定方法は 2 つあります：

1. グローバルに `MeshConfig` オプションで設定

1. 各 Pod のアノテーションで、特定のワークロードごとにカスタマイズ

{{< warning >}}
新しいトレース設定を Pod に反映させるには、Istio プロキシがインジェクトされた Pod を再起動する必要があります。

トレース設定のために追加した Pod アノテーションはグローバル設定を上書きします。グローバル設定を維持したい場合は、グローバルメッシュ設定から Pod アノテーションにコピーし、ワークロード固有のカスタマイズを行ってください。
特に、アノテーションには常にトレースバックエンドのアドレスを指定し、ワークロードのトレースが正しくレポートされるようにしてください。
{{< /warning >}}

## インストール {#installation}

これらの機能を使うことで、環境内でのトレース管理の新たな可能性が広がります。

この例では、すべてのトレースをサンプリングし、`clusterID` という名前のタグを追加します。
このタグは Pod の `ISTIO_META_CLUSTER_ID` 環境変数（最初の 256 文字のみ）から注入されます。

{{< text bash >}}
$ cat <<EOF > ./tracing.yaml
apiVersion: install.istio.io/v1alpha1
kind: IstioOperator
spec:
meshConfig:
enableTracing: true
defaultConfig:
tracing:
sampling: 100.0
max_path_tag_length: 256
custom_tags:
clusterID:
environment:
name: ISTIO_META_CLUSTER_ID
EOF
$ istioctl install -f ./tracing.yaml
{{< /text >}}

### MeshConfig でのトレース設定 {#using-mesh-config-for-trace-settings}

すべてのトレースオプションは `MeshConfig` のグローバル設定で指定できます。
設定を簡単にするため、`istioctl install -f` コマンドに渡せる YAML ファイルを作成することを推奨します。

{{< text yaml >}}
cat <<'EOF' > tracing.yaml
apiVersion: install.istio.io/v1alpha1
kind: IstioOperator
spec:
meshConfig:
enableTracing: true
defaultConfig:
tracing:
sampling: 10
custom_tags:
my_tag_header:
header:
name: host
EOF
{{< /text >}}

### `proxy.istio.io/config` アノテーションによるトレース設定 {#using-proxy-istio-io-config-annotation-for-trace-settings}

Pod のメタデータ仕様に `proxy.istio.io/config` アノテーションを追加することで、メッシュ全体のトレース設定を上書きできます。
例えば、Istio に付属の `curl` Deployment を変更するには、`samples/curl/curl.yaml` に以下を追加します：

{{< text yaml >}}
apiVersion: apps/v1
kind: Deployment
metadata:
name: curl
spec:
...
template:
metadata:
...
annotations:
...
proxy.istio.io/config: |
tracing:
sampling: 10
custom_tags:
my_tag_header:
header:
name: host
spec:
...
{{< /text >}}

## カスタマイズ {#customization}

### トレースサンプリングのカスタマイズ {#customizing-trace-sampling}

サンプリングレートオプションは、トレースシステムにレポートされるリクエストの割合を制御するために使用します。
これは、メッシュ内の通信量や収集したいトレースデータ量に応じて設定してください。
デフォルト値は 1% です。

{{< warning >}}
以前は、メッシュセットアップ時に `values.pilot.traceSampling` を変更したり、
istiod Deployment の `PILOT_TRACE_SAMPLE` 環境変数を変更する方法が推奨されていました。

この方法も引き続き有効ですが、今後は以下の方法を強く推奨します。
{{< /warning >}}

デフォルトのランダムサンプリングを 50 に変更するには、`tracing.yaml` ファイルに以下を追加します：

{{< text yaml >}}
apiVersion: install.istio.io/v1alpha1
kind: IstioOperator
spec:
meshConfig:
enableTracing: true
defaultConfig:
tracing:
sampling: 50
{{< /text >}}

サンプリングレートは 0.0 から 100.0 の範囲で、精度は 0.01 です。
例えば、1 万リクエストごとに 5 件だけトレースしたい場合は、ここに 0.05 を指定します。

### トレースタグのカスタマイズ {#customizing-tracing-tags}

span にカスタムタグを追加することで、文字列、環境変数、クライアントリクエストヘッダーなどに基づき、span に環境固有の追加情報を提供できます。

{{< warning >}}
追加できるカスタムタグの数に制限はありませんが、タグ名は一意でなければなりません。
{{< /warning >}}

以下の 3 つのサポートされているオプションのいずれかを使ってタグをカスタマイズできます。

1.  Literal は、各 span に静的な値を追加します。

    {{< text yaml >}}
    apiVersion: install.istio.io/v1alpha1
    kind: IstioOperator
    spec:
    meshConfig:
    enableTracing: true
    defaultConfig:
    tracing:
    custom_tags:
    my_tag_literal:
    literal:
    value: <VALUE>
    {{< /text >}}

1.  ワークロードプロキシの環境変数からカスタムタグの値を設定する場合は、環境変数を使用します。

    {{< text yaml >}}
    apiVersion: install.istio.io/v1alpha1
    kind: IstioOperator
    spec:
    meshConfig:
    enableTracing: true
    defaultConfig:
    tracing:
    custom_tags:
    my_tag_env:
    environment:
    name: <ENV_VARIABLE_NAME>
    defaultValue: <VALUE> # オプション
    {{< /text >}}

    {{< warning >}}
    環境変数ベースのカスタムタグを追加するには、
    Istio システムのルート名前空間にある `istio-sidecar-injector` ConfigMap を変更する必要があります。
    {{< /warning >}}

1.  クライアントリクエストヘッダーオプションは、受信クライアントリクエストヘッダーからタグ値を設定するために使用します。

    {{< text yaml >}}
    apiVersion: install.istio.io/v1alpha1
    kind: IstioOperator
    spec:
    meshConfig:
    enableTracing: true
    defaultConfig:
    tracing:
    custom_tags:
    my_tag_header:
    header:
    name: <CLIENT-HEADER>
    defaultValue: <VALUE> # オプション
    {{< /text >}}

### トレースタグ長のカスタマイズ {#customizing-tracing-tag-length}

デフォルトでは、`HttpUrl` span タグに含まれるリクエストパスの最大長は 256 です。
この最大長を変更するには、`tracing.yaml` ファイルに以下を追加してください。

{{< text yaml >}}
apiVersion: install.istio.io/v1alpha1
kind: IstioOperator
spec:
meshConfig:
enableTracing: true
defaultConfig:
tracing:
max_path_tag_length: <VALUE>
{{< /text >}}
