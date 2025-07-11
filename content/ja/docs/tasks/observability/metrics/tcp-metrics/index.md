---
title: TCP サービスメトリクスの収集
description: このタスクでは、Istio で TCP サービスのメトリクス収集を設定する方法を紹介します。
weight: 20
keywords: [telemetry, metrics, tcp]
aliases:
  - /zh/docs/tasks/telemetry/tcp-metrics
  - /zh/docs/tasks/telemetry/metrics/tcp-metrics/
owner: istio/wg-policies-and-telemetry-maintainers
test: yes
---

このタスクでは、Istio を設定してメッシュ内の TCP サービスのテレメトリデータを自動収集する方法を紹介します。
タスクの最後では、メッシュ内の TCP サービスに新しいメトリクスを有効化します。

この例では [Bookinfo](/zh/docs/examples/bookinfo/) をサンプルアプリケーションとして使用します。

## 始める前に {#before-you-begin}

- クラスタに[Istio をインストール](/zh/docs/setup/)し、アプリケーションをデプロイしてください。
  [Prometheus](/zh/docs/ops/integrations/prometheus/) もインストールする必要があります。

- このタスクでは Bookinfo アプリが `default` 名前空間にデプロイされていると仮定します。異なる名前空間を使う場合は、
  サンプルの設定やコマンドを適宜修正してください。

## 新しいテレメトリデータの収集 {#collecting-new-telemetry-data}

1. Bookinfo で MongoDB を利用するように設定します。

   1. `ratings` サービスの `v2` バージョンをインストールします。

      {{< text bash >}}
      $ kubectl apply -f @samples/bookinfo/platform/kube/bookinfo-ratings-v2.yaml@
      serviceaccount/bookinfo-ratings-v2 created
      deployment.apps/ratings-v2 created
      {{< /text >}}

   1. `mongodb` サービスをインストールします：

      {{< text bash >}}
      $ kubectl apply -f @samples/bookinfo/platform/kube/bookinfo-db.yaml@
      service/mongodb created
      deployment.apps/mongodb-v1 created
      {{< /text >}}

   1. Bookinfo サンプルは各マイクロサービスの複数バージョンをデプロイするため、まず各バージョンに対応するサブセットと
      各サブセットのロードバランシング戦略を定義する DestinationRule を作成します。

      {{< text bash >}}
      $ kubectl apply -f @samples/bookinfo/networking/destination-rule-all.yaml@
      {{< /text >}}

      双方向 TLS を有効にしている場合は、次を実行してください：

      {{< text bash >}}
      $ kubectl apply -f @samples/bookinfo/networking/destination-rule-all-mtls.yaml@
      {{< /text >}}

      次のコマンドで DestinationRule を表示できます：

      {{< text bash >}}
      $ kubectl get destinationrules -o yaml
      {{< /text >}}

      VirtualService のサブセット参照は DestinationRule に依存するため、
      サブセットを参照する VirtualService を追加する前に数秒待って DestinationRule が伝播するのを待ちます。

   1. `ratings` および `reviews` の VirtualService を作成します：

      {{< text bash >}}
      $ kubectl apply -f @samples/bookinfo/networking/virtual-service-ratings-db.yaml@
      virtualservice.networking.istio.io/reviews created
      virtualservice.networking.istio.io/ratings created
      {{< /text >}}

1. アプリケーションにトラフィックを送信します。

   Bookinfo アプリの場合、ブラウザで `http://$GATEWAY_URL/productpage` にアクセスするか、
   次のコマンドを実行します：

   {{< text bash >}}
   $ curl http://"$GATEWAY_URL"/productpage
   {{< /text >}}

   {{< tip >}}
   `$GATEWAY_URL` は [Bookinfo](/zh/docs/examples/bookinfo/) サンプルで設定した値です。
   {{< /tip >}}

1. TCP メトリクスが生成・収集されているか確認します。

   Kubernetes 環境では、次のコマンドで Prometheus のポートフォワードを設定します：

   {{< text bash >}}
   $ istioctl dashboard prometheus
   {{< /text >}}

   Prometheus のブラウザウィンドウで TCP メトリクス値を確認します。**Graph** を選択し、
   `istio_tcp_connections_opened_total` または `istio_tcp_connections_closed_total` を入力し、
   **Execute** を選択します。**Console** タブに表示されるテーブルは次のようになります：

   {{< text plain >}}
   istio_tcp_connections_opened_total{
   destination_version="v1",
   instance="172.17.0.18:42422",
   job="istio-mesh",
   canonical_service_name="ratings-v2",
   canonical_service_revision="v2"}
   {{< /text >}}

   {{< text plain >}}
   istio_tcp_connections_closed_total{
   destination_version="v1",
   instance="172.17.0.18:42422",
   job="istio-mesh",
   canonical_service_name="ratings-v2",
   canonical_service_revision="v2"}
   {{< /text >}}

## TCP テレメトリ収集の仕組み {#understanding-tcp-telemetry-collection}

このタスクでは、Istio の設定によりメッシュ内の TCP サービスの全トラフィックについて自動的にメトリクスが生成・レポートされます。
デフォルトでは、すべてのアクティブな接続の TCP メトリクスは `15s` ごとに記録され、このタイマーは
`tcpReportingDuration` で設定できます。接続終了時にもメトリクスが記録されます。

### TCP 属性 {#tcp-attributes}

TCP 固有の属性がいくつかあり、Istio で TCP ポリシーや制御を有効化できます。これらの属性は Envoy
プロキシによって生成され、Envoy の Node Metadata を通じて Istio から取得されます。Envoy は
ALPN ベースのトンネリングやプレフィックスベースのプロトコルを使ってノードメタデータをピア Envoy に転送します。
新しいプロトコル `istio-peer-exchange` を定義しており、これはメッシュ内のクライアントと Sidecar サーバーのアドバタイズと優先度を定義します。
Istio 間の接続で有効な場合、ALPN ネゴシエーションによりプロトコルが `istio-peer-exchange` プロキシに解決され、
Istio のプロキシや他のプロキシは有効になりません。このプロトコルは TCP を次のように拡張します：

1. TCP クライアントは、最初のバイト列としてマジックバイト列と長さ付きペイロードを送信します。
1. TCP サーバーも、最初のバイト列としてマジックバイト列と長さ付きペイロード（protobuf でエンコードされたシリアライズメタデータ）を送信します。
1. クライアントとサーバーは同時に書き込みでき、順序は混在します。Envoy の拡張フィルタが下流・上流で処理し、
   マジックバイト列が一致しないか、ペイロード全体を読み込むまで続きます。

{{< image link="./alpn-based-tunneling-protocol.svg"
    alt="Istio サービスメッシュにおける TCP サービス属性生成フロー"
    caption="TCP 属性フロー"
    >}}

## クリーンアップ {#cleanup}

- `port-forward` プロセスを削除します：

  {{< text bash >}}
  $ killall istioctl
  {{< /text >}}

- 他のタスクを試す予定がなければ、[Bookinfo のクリーンアップ](/zh/docs/examples/bookinfo/#cleanup)に従い、サンプルアプリケーションを削除してください。
