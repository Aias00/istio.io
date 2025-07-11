---
title: Apache SkyWalking
description: プロキシを設定して Apache SkyWalking へトレースリクエストを送信する方法を学びます。
weight: 8
keywords: [telemetry, tracing, skywalking, span, port-forwarding]
owner: istio/wg-policies-and-telemetry-maintainers
test: yes
---

このタスクを完了すると、[Apache SkyWalking](https://skywalking.apache.org) を使ってアプリケーションをトレースする方法が分かります。アプリケーションの言語、フレームワーク、プラットフォームは問いません。

このタスクでは [Bookinfo](/zh/docs/examples/bookinfo/) サンプルアプリケーションを使用します。

Istio がどのようにトレースを処理するかについては、[分散トレースの概要](../overview/)を参照してください。

## トレースの設定 {#configure-tracing}

`IstioOperator` 設定で Istio をインストールする場合、以下のフィールドを設定に追加してください：

{{< text yaml >}}
apiVersion: install.istio.io/v1alpha1
kind: IstioOperator
spec:
meshConfig:
defaultProviders:
tracing: - "skywalking"
enableTracing: true
extensionProviders: - name: "skywalking"
skywalking:
service: tracing.istio-system.svc.cluster.local
port: 11800
{{< /text >}}

この設定で Istio をインストールすると、SkyWalking Agent がデフォルトのトレーサーとして使用され、トレースデータは SkyWalking バックエンドに送信されます。

デフォルト設定ではサンプリング率は 1% です。
[Telemetry API](/zh/docs/tasks/observability/telemetry/) を使って 100% に引き上げるには：

{{< text bash >}}
$ kubectl apply -f - <<EOF
apiVersion: telemetry.istio.io/v1
kind: Telemetry
metadata:
name: mesh-default
namespace: istio-system
spec:
tracing:

- randomSamplingPercentage: 100.00
  EOF
  {{< /text >}}

## SkyWalking コレクターのデプロイ {#deploy-skywalking-collector}

[SkyWalking インストール](/zh/docs/ops/integrations/skywalking/#installation)ドキュメントに従い、
SkyWalking をクラスタにデプロイしてください。

## Bookinfo アプリケーションのデプロイ {#deploy-bookinfo-app}

[Bookinfo](/zh/docs/examples/bookinfo/#deploying-the-application) サンプルアプリケーションをデプロイします。

## ダッシュボードへのアクセス {#accessing-dashboard}

[リモートでのテレメトリプラグインへのアクセス](/zh/docs/tasks/observability/gateways)タスクでは、Gateway 経由で Istio プラグインへアクセスする方法を詳しく説明しています。

テスト（または一時的なアクセス）の場合、ポートフォワーディングも利用できます。SkyWalking が `istio-system` 名前空間にデプロイされていると仮定し、以下を使用してください：

{{< text bash >}}
$ istioctl dashboard skywalking
{{< /text >}}

## Bookinfo サンプルでトレースを生成する {#generating-tarces-using-bookinfo}

1.  Bookinfo アプリケーションが起動し稼働している状態で、`http://$GATEWAY_URL/productpage` に一度または複数回アクセスしてトレース情報を生成します：

    {{< boilerplate trace-generation >}}

1.  "General Service" パネルからサービス一覧を確認できます：

    {{< image link="./istio-service-list-skywalking.png" caption="Service List" >}}

1.  メインコンテンツで `Trace` タブを選択します。左側のリストでトレース一覧、右側のパネルでトレース詳細が確認できます：

    {{< image link="./istio-tracing-list-skywalking.png" caption="Trace View" >}}

1.  トレースは複数の span で構成されており、それぞれが `/productpage` 実行時に呼び出される Bookinfo サービスや、`istio-ingressgateway` などの Istio 内部コンポーネントに対応します。

## SkyWalking 公式デモアプリの探索 {#explore-skywalking-official-demo-app}

このチュートリアルでは [Bookinfo](/zh/docs/examples/bookinfo/#deploying-the-application) サンプルアプリケーションを使用しています。
このサンプルアプリでは、サービスに SkyWalking エージェントはインストールされておらず、すべてのトレースは Sidecar プロキシによって生成されます。

[SkyWalking 言語エージェント](https://skywalking.apache.org/docs/#Agent) についてさらに知りたい場合は、
SkyWalking チームが提供する[デモアプリ](http://github.com/apache/skywalking-showcase)もご覧ください。
ここでは、より詳細なトレースや、言語エージェント固有の機能（例：プロファイル分析）も体験できます。

## クリーンアップ {#cleanup}

1.  Ctrl-C を使うか、実行中の `istioctl` プロセスをすべて終了してください：

    {{< text bash >}}
    $ killall istioctl
    {{< /text >}}

1.  今後のタスクを試す予定がなければ、[Bookinfo のクリーンアップ](/zh/docs/examples/bookinfo/#cleanup)の手順に従ってアプリケーション全体を停止してください。
