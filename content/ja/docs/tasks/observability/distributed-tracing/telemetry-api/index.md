---
title: Telemetry API でトレースを設定する
description: Telemetry API を使ってトレースを設定する方法。
weight: 2
keywords: [telemetry, tracing]
owner: istio/wg-policies-and-telemetry-maintainers
test: yes
---

Istio では、サンプリングレートやレポートされる Span へのカスタムタグ追加など、トレースオプションの設定機能が提供されています。
このタスクでは、Telemetry API を使ってトレースオプションをカスタマイズする方法を紹介します。

## 始める前に {#before-you-begin}

1. アプリケーションが[こちら](/zh/docs/tasks/observability/distributed-tracing/overview/)で説明されているようにトレースヘッダーを設定していることを確認してください。

1. 希望するトレースバックエンドに応じて、[統合](/zh/docs/ops/integrations/)の下にあるトレースインストールガイドに従い、適切なソフトウェアをインストールし、拡張プロバイダーを設定してください。

## インストール {#installation}

この例では、トレースを [Zipkin](/zh/docs/ops/integrations/zipkin/) に送信します。
続行する前に Zipkin をインストールしてください。

### 拡張プロバイダーの設定 {#configure-an-extension-provider}

Zipkin サービスを参照する[拡張プロバイダー](/zh/docs/reference/config/istio.mesh.v1alpha1/#MeshConfig-ExtensionProvider)を使って Istio をインストールします：

{{< text bash >}}
$ cat <<EOF > ./tracing.yaml
apiVersion: install.istio.io/v1alpha1
kind: IstioOperator
spec:
meshConfig:
enableTracing: true
defaultConfig:
tracing: {} # 旧 MeshConfig トレースオプションを無効化
extensionProviders: # zipkin プロバイダーを追加 - name: "zipkin"
zipkin:
service: zipkin.istio-system.svc.cluster.local
port: 9411
EOF
$ istioctl install -f ./tracing.yaml --skip-confirmation
{{< /text >}}

### トレースの有効化 {#enable-tracing}

以下の設定を適用してトレースを有効にします：

{{< text bash >}}
$ kubectl apply -f - <<EOF
apiVersion: telemetry.istio.io/v1
kind: Telemetry
metadata:
name: mesh-default
namespace: istio-system
spec:
tracing:

- providers: - name: "zipkin"
  EOF
  {{< /text >}}

### 結果の検証 {#verify-the-results}

[Zipkin UI にアクセス](/zh/docs/tasks/observability/distributed-tracing/zipkin/)して結果を検証できます。

## カスタマイズ {#customization}

### トレースサンプリングのカスタマイズ {#customizing-trace-sampling}

サンプリングレートオプションは、トレースシステムにレポートされるリクエストの割合を制御するために使用します。
これは、サービスメッシュ内のトラフィックや収集したいトレースデータ量に応じて設定してください。
デフォルト値は 1% です。

{{< text bash >}}
$ kubectl apply -f - <<EOF
apiVersion: telemetry.istio.io/v1
kind: Telemetry
metadata:
name: mesh-default
namespace: istio-system
spec:
tracing:

- providers: - name: "zipkin"
  randomSamplingPercentage: 100.00
  EOF
  {{< /text >}}

### トレースタグのカスタマイズ {#customizing-tracing-tags}

span にカスタムタグを追加することで、文字列、環境変数、クライアントリクエストヘッダーなどに基づき、span に環境固有の追加情報を提供できます。

{{< warning >}}
追加できるカスタムタグの数に制限はありませんが、タグ名は一意でなければなりません。
{{< /warning >}}

以下の 3 つの方法でカスタムタグを追加できます。

1.  literal オプションは、各 span に静的な値を追加します。

    {{< text yaml >}}
    apiVersion: telemetry.istio.io/v1
    kind: Telemetry
    metadata:
    name: mesh-default
    namespace: istio-system
    spec:
    tracing:

    - providers: - name: "zipkin"
      randomSamplingPercentage: 100.00
      customTags:
      "provider":
      literal:
      value: "zipkin"
      {{< /text >}}

1.  環境変数を使ってワークロードプロキシの環境からカスタムタグを設定できます。

    {{< text yaml >}}
    apiVersion: telemetry.istio.io/v1
    kind: Telemetry
    metadata:
    name: mesh-default
    namespace: istio-system
    spec:
    tracing: - providers: - name: "zipkin"
    randomSamplingPercentage: 100.00
    customTags:
    "cluster_id":
    environment:
    name: ISTIO_META_CLUSTER_ID
    defaultValue: Kubernetes # オプション
    {{< /text >}}

    {{< warning >}}
    環境変数ベースのカスタムタグを追加するには、
    Istio システムのルート名前空間にある `istio-sidecar-injector` の ConfigMap を変更する必要があります。
    {{< /warning >}}

1.  クライアントリクエストヘッダーオプションは、受信クライアントリクエストヘッダーからタグ値を設定するために使用します。

    {{< text yaml >}}
    apiVersion: telemetry.istio.io/v1
    kind: Telemetry
    metadata:
    name: mesh-default
    namespace: istio-system
    spec:
    tracing: - providers: - name: "zipkin"
    randomSamplingPercentage: 100.00
    customTags:
    my_tag_header:
    header:
    name: <CLIENT-HEADER>
    defaultValue: <VALUE> # オプション
    {{< /text >}}

### トレースタグ長のカスタマイズ {#customizing-tracing-tag-length}

デフォルトでは、`HttpUrl` span タグに含まれるリクエストパスの最大長は 256 です。この最大長を変更するには、`tracing.yaml` 設定ファイルに以下を追加してください。

{{< text yaml >}}
apiVersion: install.istio.io/v1alpha1
kind: IstioOperator
spec:
meshConfig:
enableTracing: true
defaultConfig:
tracing: {} # MeshConfig で旧トレースオプションを無効化
extensionProviders: - name: "zipkin"
zipkin:
service: zipkin.istio-system.svc.cluster.local
port: 9411
maxTagLength: <VALUE>
{{< /text >}}
