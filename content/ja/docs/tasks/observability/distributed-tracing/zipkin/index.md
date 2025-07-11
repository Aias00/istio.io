---
title: Zipkin
description: プロキシを設定して Zipkin へトレースリクエストを送信する方法を学びます。
weight: 7
keywords: [telemetry, tracing, zipkin, span, port-forwarding]
aliases:
  - /zh/docs/tasks/zipkin-tracing.html
owner: istio/wg-policies-and-telemetry-maintainers
test: yes
---

このタスクを通じて、[Zipkin](https://zipkin.io/) でアプリケーションをトレースする方法が分かります。
アプリケーションの開発言語、フレームワーク、プラットフォームは問いません。

このタスクでは [Bookinfo](/zh/docs/examples/bookinfo/) サンプルアプリケーションを使用します。

Istio がどのようにトレースを処理するかについては、このタスクの[概要](../overview/)をご覧ください。

## 始める前に {#before-you-begin}

1. [Zipkin インストール](/zh/docs/setup/install/istioctl)ドキュメントに従い、Zipkin をクラスタにインストールしてください。

1. [Bookinfo](/zh/docs/examples/bookinfo/#deploying-the-application) サンプルアプリケーションをデプロイしてください。

## Istio の分散トレース設定 {#configure-istio-for-distributed-tracing}

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
extensionProviders: - name: zipkin
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

- providers: - name: zipkin
  EOF
  {{< /text >}}

## ダッシュボードへのアクセス {#accessing-the-dashboard}

[リモートでのテレメトリプラグインへのアクセス](/zh/docs/tasks/observability/gateways)タスクでは、ゲートウェイ経由で Istio プラグインへアクセスする方法を詳しく説明しています。

テスト（または一時的なアクセス）の場合、ポートフォワーディングも利用できます。Zipkin が `istio-system` 名前空間にデプロイされていると仮定し、以下を使用してください：

{{< text bash >}}
$ istioctl dashboard zipkin
{{< /text >}}

## Bookinfo サンプルでトレースを生成する {#generating-traces-using-the-Bookinfo-sample}

1. Bookinfo アプリケーションが起動し稼働している状態で、`http://$GATEWAY_URL/productpage` に一度または複数回アクセスしてトレース情報を生成します。

   {{< boilerplate trace-generation >}}

1. 検索パネルで `+` をクリックし、最初のドロップダウンから `serviceName` を選択し、
   2 番目のドロップダウンから `productpage.default` を選択して検索アイコンをクリックします：

   {{< image link="./istio-tracing-list-zipkin.png" caption="Tracing Dashboard" >}}

1. `ISTIO-INGRESSGATEWAY` の検索結果をクリックし、最新の `/productpage` リクエストの詳細を確認します：

   {{< image link="./istio-tracing-details-zipkin.png" caption="Detailed Trace View" >}}

1. トレースは複数の Span で構成されており、それぞれが `/productpage` リクエストや Istio の内部コンポーネント（例：`istio-ingressgateway`）で呼び出される Bookinfo サービスに対応します。

## クリーンアップ {#cleanup}

1. Control-C を使うか、実行中の `istioctl` プロセスをすべて終了してください：

   {{< text bash >}}
   $ killall istioctl
   {{< /text >}}

1. 今後のタスクを試す予定がなければ、[Bookinfo のクリーンアップ](/zh/docs/examples/bookinfo/#cleanup)の手順に従ってアプリケーション全体を停止してください。
