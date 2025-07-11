---
title: Bookinfo アプリケーション
description: 複数の Istio 機能をデモするためのサンプルアプリケーションをデプロイします。4 つの独立したマイクロサービスで構成されています。
weight: 10
aliases:
  - /zh/docs/samples/bookinfo.html
  - /zh/docs/guides/bookinfo/index.html
  - /zh/docs/guides/bookinfo.html
owner: istio/wg-docs-maintainers
test: yes
---

このサンプルは、複数の Istio 機能をデモするためのアプリケーションをデプロイします。このアプリケーションは 4 つの独立したマイクロサービスで構成されています。

{{< tip >}}
[はじめに](/ja/docs/setup/getting-started/)ガイドに従って Istio をインストールした場合、すでに Bookinfo がインストールされています。以下のほとんどの手順をスキップし、[サービスバージョンの定義](/ja/docs/examples/bookinfo/#define-the-service-versions)に進んでください。
{{< /tip >}}

Bookinfo アプリケーションは、書籍の情報ページ（オンライン書店のカテゴリページのようなもの）を表示します。このページには書籍の説明、詳細（ISBN、ページ数など）、およびその書籍に関連するいくつかのレビューが表示されます。

Bookinfo アプリケーションは 4 つの独立したマイクロサービスに分割されています：

- `productpage`：このマイクロサービスは `details` と `reviews` の 2 つのマイクロサービスを呼び出し、ページの内容を埋めます。
- `details`：このマイクロサービスには書籍の詳細情報が含まれます。
- `reviews`：このマイクロサービスには書籍のレビューが含まれ、`ratings` マイクロサービスも呼び出します。
- `ratings`：このマイクロサービスには書籍レビューの評価情報が含まれます。

`reviews` マイクロサービスには 3 つのバージョンがあります：

- v1 バージョンは `ratings` サービスを呼び出しません。
- v2 バージョンは `ratings` サービスを呼び出し、1 ～ 5 個の黒い星アイコンで評価を表示します。
- v3 バージョンは `ratings` サービスを呼び出し、1 ～ 5 個の赤い星アイコンで評価を表示します。

下図はこのアプリケーションのエンドツーエンドのアーキテクチャを示しています。

{{< image width="80%" link="./noistio.svg" caption="Istio 未使用の Bookinfo アプリケーション" >}}

Bookinfo アプリケーションのいくつかのマイクロサービスは異なるプログラミング言語で実装されています。これらのサービスは Istio への依存はありませんが、代表的なサービスメッシュの例となっています：複数のサービス、複数の言語、複数バージョンの `reviews` サービスを含みます。

## はじめに {#before-you-begin}

まだ始めていない場合は、[インストールガイド](/ja/docs/setup/)に従って Istio をデプロイしてください。

{{< boilerplate gateway-api-support >}}

## アプリケーションのデプロイ {#deploying-the-application}

Istio でこのサンプルアプリケーションを実行するには、アプリケーション自体を変更する必要はありません。Istio が有効な環境でサービスを設定・実行し、各サービスに Envoy Sidecar を注入するだけです。最終的なデプロイは下図のようになります：

{{< image width="80%" link="./withistio.svg" caption="Bookinfo アプリケーション" >}}

すべてのマイクロサービスは Envoy Sidecar と統合され、統合されたサービスのすべての入出力トラフィックは Sidecar によってインターセプトされます。これにより、外部制御のためのフックが提供され、Istio コントロールプレーンを使ってアプリ全体にサービスルーティング、テレメトリ収集、ポリシー適用などの機能を提供できます。

### アプリケーションサービスの起動 {#start-the-application-services}

{{< tip >}}
GKE を使用している場合、クラスタに少なくとも 4 つの標準 GKE ノードがあることを確認してください。Minikube の場合は 4GB 以上のメモリが必要です。
{{< /tip >}}

1. Istio インストールディレクトリに移動します。

1. Istio ではデフォルトで[Sidecar の自動注入](/ja/docs/setup/additional-setup/sidecar-injection/#automatic-sidecar-injection)が有効です。
   `default` 名前空間に `istio-injection=enabled` ラベルを付与します：

   {{< text bash >}}
   $ kubectl label namespace default istio-injection=enabled
   {{< /text >}}

1. `kubectl` コマンドでアプリケーションをデプロイします：

   {{< text bash >}}
   $ kubectl apply -f @samples/bookinfo/platform/kube/bookinfo.yaml@
   {{< /text >}}

   上記コマンドは、Bookinfo アプリケーションアーキテクチャ図に示されている 4 つのサービスすべてを起動します。また、reviews サービスの 3 つのバージョン（v1、v2、v3）も起動します。

   {{< tip >}}
   実際のデプロイでは、すべてのバージョンを同時にデプロイするのではなく、新しいバージョンのマイクロサービスを順次デプロイします。
   {{< /tip >}}

1. すべてのサービスと Pod が正しく定義・起動されていることを確認します：

   {{< text bash >}}
   $ kubectl get services
   NAME TYPE CLUSTER-IP EXTERNAL-IP PORT(S) AGE
   details ClusterIP 10.0.0.31 <none> 9080/TCP 6m
   kubernetes ClusterIP 10.0.0.1 <none> 443/TCP 7d
   productpage ClusterIP 10.0.0.120 <none> 9080/TCP 6m
   ratings ClusterIP 10.0.0.15 <none> 9080/TCP 6m
   reviews ClusterIP 10.0.0.170 <none> 9080/TCP 6m
   {{< /text >}}

   また：

   {{< text bash >}}
   $ kubectl get pods
   NAME READY STATUS RESTARTS AGE
   details-v1-1520924117-48z17 2/2 Running 0 6m
   productpage-v1-560495357-jk1lz 2/2 Running 0 6m
   ratings-v1-734492171-rnr5l 2/2 Running 0 6m
   reviews-v1-874083890-f0qf0 2/2 Running 0 6m
   reviews-v2-1343845940-b34q5 2/2 Running 0 6m
   reviews-v3-1813607990-8ch52 2/2 Running 0 6m
   {{< /text >}}

1. Bookinfo アプリケーションが稼働していることを確認するには、いずれかの Pod（例：`ratings`）から `curl` コマンドでリクエストを送信します：

   {{< text bash >}}
   $ kubectl exec "$(kubectl get pod -l app=ratings -o jsonpath='{.items[0].metadata.name}')" -c ratings -- curl -sS productpage:9080/productpage | grep -o "<title>.\*</title>"
   <title>Simple Bookstore App</title>
   {{< /text >}}

### Ingress の IP とポートの確認 {#determine-the-ingress-IP-and-port}

Bookinfo サービスが起動したら、Kubernetes クラスタ外部からアクセスできるようにする必要があります（例：ブラウザからアクセス）。これにはゲートウェイを使用します。

1. Bookinfo アプリケーション用のゲートウェイを定義します：

   {{< tabset category-name="config-api" >}}

   {{< tab name="Istio APIs" category-value="istio-apis" >}}

   次のコマンドで [Istio ゲートウェイ](/ja/docs/concepts/traffic-management/#gateways)を作成します：

   {{< text bash >}}
   $ kubectl apply -f @samples/bookinfo/networking/bookinfo-gateway.yaml@
   gateway.networking.istio.io/bookinfo-gateway created
   virtualservice.networking.istio.io/bookinfo created
   {{< /text >}}

   ゲートウェイが作成されたことを確認します：

   {{< text bash >}}
   $ kubectl get gateway
   NAME AGE
   bookinfo-gateway 32s
   {{< /text >}}

   [こちらの手順](/ja/docs/tasks/traffic-management/ingress/ingress-control/#determining-the-ingress-ip-and-ports)に従って `INGRESS_HOST` と `INGRESS_PORT` 変数を設定し、戻ってきてください。

   {{< /tab >}}

   {{< tab name="Gateway API" category-value="gateway-api" >}}

   {{< boilerplate external-loadbalancer-support >}}

   次のコマンドで [Kubernetes Gateway](https://gateway-api.sigs.k8s.io/api-types/gateway/) を作成します：

   {{< text bash >}}
   $ kubectl apply -f @samples/bookinfo/gateway-api/bookinfo-gateway.yaml@
   gateway.gateway.networking.k8s.io/bookinfo-gateway created
   httproute.gateway.networking.k8s.io/bookinfo created
   {{< /text >}}

   Kubernetes `Gateway` リソースの作成により[関連するプロキシサービスもデプロイ](/ja/docs/tasks/traffic-management/ingress/gateway-api/#automated-deployment)されるため、次のコマンドでゲートウェイの準備完了を待ちます：

   {{< text bash >}}
   $ kubectl wait --for=condition=programmed gtw bookinfo-gateway
   {{< /text >}}

   Bookinfo ゲートウェイリソースからゲートウェイのアドレスとポートを取得します：

   {{< text bash >}}
   $ export INGRESS_HOST=$(kubectl get gtw bookinfo-gateway -o jsonpath='{.status.addresses[0].value}')
    $ export INGRESS_PORT=$(kubectl get gtw bookinfo-gateway -o jsonpath='{.spec.listeners[?(@.name=="http")].port}')
   {{< /text >}}

   {{< /tab >}}

   {{< /tabset >}}

1. `GATEWAY_URL` を設定します：

   {{< text bash >}}
   $ export GATEWAY_URL=$INGRESS_HOST:$INGRESS_PORT
   {{< /text >}}

## クラスタ外部からアプリケーションにアクセスできることの確認 {#confirm-the-app-is-accessible-from-outside-the-cluster}

Bookinfo アプリケーションにクラスタ外部からアクセスできるかどうかを確認するには、次の `curl` コマンドを実行します：

{{< text bash >}}
$ curl -s "http://${GATEWAY_URL}/productpage" | grep -o "<title>.\*</title>"

<title>Simple Bookstore App</title>
{{< /text >}}

また、ブラウザで `http://$GATEWAY_URL/productpage` を開いてアプリの Web ページを表示できます。ページを何度かリロードすると、`productpage` ページで `reviews` サービスの異なるバージョン（赤い星、黒い星、星なし）がラウンドロビンで表示されるのが分かります。これは、まだ Istio でバージョンルーティングを制御していないためです。

## サービスバージョンの定義 {#define-the-service-versions}

Istio で Bookinfo のバージョンルーティングを制御する前に、利用可能なバージョンを定義する必要があります。

{{< tabset category-name="config-api" >}}

{{< tab name="Istio APIs" category-value="istio-apis" >}}

Istio では[目標ルール](/ja/docs/concepts/traffic-management/#destination-rules)でサービスのバージョンを **subsets（サブセット）** として定義します。次のコマンドで Bookinfo サービスのデフォルト目標ルールを作成します：

{{< text bash >}}
$ kubectl apply -f @samples/bookinfo/networking/destination-rule-all.yaml@
{{< /text >}}

{{< tip >}}
`default` および `demo` の[構成ファイル](/ja/docs/setup/additional-setup/config-profiles/)では[自動双方向 TLS](/ja/docs/tasks/security/authentication/authn-policy/#auto-mutual-tls)がデフォルトで有効です。双方向 TLS を強制するには、`samples/bookinfo/networking/destination-rule-all-mtls.yaml` の目標ルールを使用してください。
{{< /tip >}}

数秒待って目標ルールが有効になるのを待ちます。

次のコマンドで目標ルールを確認できます：

{{< text bash >}}
$ kubectl get destinationrules -o yaml
{{< /text >}}

{{< /tab >}}

{{< tab name="Gateway API" category-value="gateway-api" >}}

Istio API で `DestinationRule` サブセットを使ってサービスのバージョンを定義するのとは異なり、Kubernetes Gateway API ではバックエンドサービス定義を使います。

次のコマンドで 3 つの `reviews` サービスバージョンのバックエンドサービス定義を作成します：

{{< text bash >}}
$ kubectl apply -f @samples/bookinfo/platform/kube/bookinfo-versions.yaml@
{{< /text >}}

{{< /tab >}}

{{< /tabset >}}

## 次のステップ {#whats-next}

このアプリケーションを使って、Istio のトラフィックルーティング、フォールトインジェクション、レートリミットなどの機能を体験できます。次は[Istio タスク](/ja/docs/tasks)のいずれかを読んで試してみてください。初心者には[リクエストルーティングの設定](/ja/docs/tasks/traffic-management/request-routing/)から始めるのがおすすめです。

## クリーンアップ {#cleanup}

Bookinfo サンプルアプリケーションの体験が終わったら、次のコマンドでアプリの削除とクリーンアップができます：

{{< text bash >}}
$ @samples/bookinfo/platform/kube/cleanup.sh@
{{< /text >}}
