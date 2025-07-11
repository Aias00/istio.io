---
title: Jaeger
description: Jaeger へトレースリクエストを送信するようにプロキシを設定する方法を学びます。
weight: 6
keywords: [telemetry, tracing, jaeger, span, port-forwarding]
aliases:
  - /zh/docs/tasks/telemetry/distributed-tracing/jaeger/
owner: istio/wg-policies-and-telemetry-maintainers
test: yes
---

このタスクを完了すると、どの言語、フレームワーク、またはプラットフォームでアプリケーションを構築していても、[Jaeger](https://www.jaegertracing.io/) のトレースにアプリケーションを参加させる方法が分かります。

このタスクでは、デモ用アプリケーションとして [Bookinfo](/zh/docs/examples/bookinfo/) を使用します。

Istio がどのようにトレースを処理するかについては、このタスクの[概要](../overview/)をご覧ください。

## 始める前に {#before-you-begin}

1. [Jaeger のインストール](/zh/docs/ops/integrations/jaeger/#installation)ドキュメントに従って、Jaeger をクラスタにインストールしてください。

1. [Bookinfo](/zh/docs/examples/bookinfo/#deploying-the-application) サンプルアプリケーションをデプロイしてください。

## Istio の分散トレース設定 {#configure-istio-for-distributed-tracing}

### 拡張プロバイダーの設定 {#configure-an-extension-provider}

Jaeger コレクターサービスを参照する[拡張プロバイダー](/zh/docs/reference/config/istio.mesh.v1alpha1/#MeshConfig-ExtensionProvider)を使って Istio をインストールします：

{{< text bash >}}
$ cat <<EOF > ./tracing.yaml
apiVersion: install.istio.io/v1alpha1
kind: IstioOperator
spec:
meshConfig:
enableTracing: true
defaultConfig:
tracing: {} # 旧 MeshConfig トレースオプションを無効化
extensionProviders: - name: jaeger
opentelemetry:
port: 4317
service: jaeger-collector.istio-system.svc.cluster.local
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

- providers: - name: jaeger
  EOF
  {{< /text >}}

## ダッシュボードへのアクセス {#accessing-the-dashboard}

[リモートでのテレメトリプラグインへのアクセス](/zh/docs/tasks/observability/gateways)タスクでは、ゲートウェイ経由で Istio プラグインへアクセスする方法を詳しく説明しています。

テスト（または一時的なアクセス）の場合、ポートフォワーディングも利用できます。Jaeger が `istio-system` 名前空間にデプロイされていると仮定し、以下を使用してください：

{{< text bash >}}
$ istioctl dashboard jaeger
{{< /text >}}

## Bookinfo サンプルでトレースを生成する{#generating-traces-using-the-Bookinfo-sample}

1. Bookinfo アプリケーションが起動し稼働している状態で、`http://$GATEWAY_URL/productpage` に一度または複数回アクセスしてトレースを生成します。

   {{< boilerplate trace-generation >}}

1. ダッシュボード左側の **Service** ドロップダウンから `productpage.default` を選択し、**Find Traces** をクリックします：

   {{< image link="./istio-tracing-list.png" caption="トレースダッシュボード" >}}

1. 一番上の最新のトレースをクリックし、直近の `/productpage` アクセスの詳細を確認します：

   {{< image link="./istio-tracing-details.png" caption="詳細なトレースビュー" >}}

1. トレースは複数の Span で構成されており、それぞれが Bookinfo サービスまたは Istio の内部コンポーネント（例：`istio-ingressgateway`）で `/productpage` リクエスト時に呼び出されます。

## クリーンアップ{#cleanup}

1. Control C を使うか、実行中の `istioctl` プロセスをすべて終了してください：

   {{< text bash >}}
   $ killall istioctl
   {{< /text >}}

1. 今後のタスクを試す予定がなければ、[Bookinfo のクリーンアップ](/zh/docs/examples/bookinfo/#cleanup)の手順に従って、アプリケーション全体を停止してください。
