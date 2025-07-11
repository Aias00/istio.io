---
title: Grafana でメトリクスを可視化する
description: このタスクでは、Istio ダッシュボードをセットアップし、メッシュトラフィックを監視する方法を紹介します。
weight: 40
keywords: [telemetry, visualization]
aliases:
  - /zh/docs/tasks/telemetry/using-istio-dashboard/
  - /zh/docs/tasks/telemetry/metrics/using-istio-dashboard/
owner: istio/wg-policies-and-telemetry-maintainers
test: yes
---

このタスクでは、Istio ダッシュボードをセットアップし、メッシュトラフィックを監視する方法を紹介します。
このタスクの一環として、Grafana の Istio アドオンと Web ベースの UI を使ってサービスメッシュのトラフィックデータを確認します。

このタスクでは [Bookinfo](/zh/docs/examples/bookinfo/) をサンプルアプリケーションとして使用します。

## 始める前に {#before-you-begin}

- クラスタに[Istio をインストール](/zh/docs/setup/)してください。
- [Grafana アドオン](/zh/docs/ops/integrations/grafana/#option-1-quick-start)をインストールしてください。
- [Prometheus アドオン](/zh//docs/ops/integrations/prometheus/#option-1-quick-start)をインストールしてください。
- [Bookinfo](/zh/docs/examples/bookinfo/) アプリケーションをデプロイしてください。

## Istio ダッシュボードの表示 {#viewing-the-Istio-dashboard}

1. `prometheus` サービスがクラスタ内で稼働していることを確認します。

   Kubernetes 環境で次のコマンドを実行します：

   {{< text bash >}}
   $ kubectl -n istio-system get svc prometheus
   NAME TYPE CLUSTER-IP EXTERNAL-IP PORT(S) AGE
   prometheus ClusterIP 10.100.250.202 <none> 9090/TCP 103s
   {{< /text >}}

1. Grafana サービスがクラスタ内で稼働していることを確認します。

   Kubernetes クラスタで次のコマンドを実行します：

   {{< text bash >}}
   $ kubectl -n istio-system get svc grafana
   NAME TYPE CLUSTER-IP EXTERNAL-IP PORT(S) AGE
   grafana ClusterIP 10.103.244.103 <none> 3000/TCP 2m25s
   {{< /text >}}

1. Grafana UI から Istio ダッシュボードを開きます。

   Kubernetes クラスタで次のコマンドを実行します：

   {{< text bash >}}
   $ istioctl dashboard grafana
   {{< /text >}}

   ブラウザで [http://localhost:3000/d/G8wLrJIZk/istio-mesh-dashboard](http://localhost:3000/d/G8wLrJIZk/istio-mesh-dashboard) にアクセスします。

   Istio ダッシュボードは次のように表示されます：

   {{< image link="./grafana-istio-dashboard.png" caption="Istio ダッシュボード" >}}

1. メッシュアプリケーションにトラフィックを送信します。

   Bookinfo サンプルの場合、ブラウザで `http://$GATEWAY_URL/productpage` にアクセスするか、次のコマンドを実行します：

   {{< boilerplate trace-generation >}}

   {{< tip >}}
   `$GATEWAY_URL` は [Bookinfo](/zh/docs/examples/bookinfo/) サンプルで設定した値です。
   {{< /tip >}}

   ページを数回リロード（またはコマンドを数回実行）して少量のトラフィックを発生させてください。

   再度 Istio ダッシュボードを確認すると、発生したトラフィックが反映され、次のように表示されます：

   {{< image link="./dashboard-with-traffic.png" caption="Istio トラフィックダッシュボード" >}}

   これにより、メッシュ全体やメッシュ内のサービス・ワークロードのグローバルビューが得られ、
   特定のダッシュボードに移動することでサービスやワークロードの詳細情報も確認できます（後述）。

1. サービスダッシュボードの可視化。

   Grafana ダッシュボード左上のナビゲーションメニューから Istio Service Dashboard に移動するか、
   ブラウザで [http://localhost:3000/d/LJ_uJAvmk/istio-service-dashboard](http://localhost:3000/d/LJ_uJAvmk/istio-service-dashboard) にアクセスします。

   {{< tip >}}
   サービスのドロップダウンリストからサービスを選択する必要がある場合があります。
   {{< /tip >}}

   Istio Service Dashboard は次のように表示されます：

   {{< image link="./istio-service-dashboard.png" caption="Istio サービスダッシュボード" >}}

   ここではサービスごと、さらにそのサービスのクライアントワークロード（そのサービスを呼び出すワークロード）や
   サービスワークロード（そのサービスを提供するワークロード）の詳細なメトリクスが表示されます。

1. ワークロードダッシュボードの可視化。

   Grafana ダッシュボード左上のナビゲーションメニューから Istio Workload Dashboard に移動するか、
   ブラウザで [http://localhost:3000/d/UbsSZTDik/istio-workload-dashboard](http://localhost:3000/d/UbsSZTDik/istio-workload-dashboard) にアクセスします。

   Istio Workload Dashboard は次のように表示されます：

   {{< image link="./istio-workload-dashboard.png" caption="Istio ワークロードダッシュボード" >}}

   ここでは各ワークロードごと、さらにそのワークロードのインバウンドワークロード（そのワークロードにリクエストを送るワークロード）
   やアウトバウンドサービス（このワークロードがリクエストを送るサービス）の詳細なメトリクスが表示されます。

### Grafana ダッシュボードについて {#about-the-Grafana-dashboards}

Istio ダッシュボードは主に 3 つのセクションで構成されています：

1. メッシュサマリービュー：メッシュ全体のグローバルサマリーを提供し、メッシュ内（HTTP/gRPC および TCP）のワークロードを表示します。

1. サービスごとのビュー：メッシュ内の各サービス（HTTP/gRPC および TCP）について、リクエスト・レスポンスのメトリクスや
   そのサービスのクライアント・サービスワークロードのメトリクスを表示します。

1. ワークロードごとのビュー：メッシュ内の各ワークロード（HTTP/gRPC および TCP）について、リクエスト・レスポンスのメトリクスや
   そのワークロードのインバウンドワークロード・アウトバウンドサービスのメトリクスを表示します。

ダッシュボードの作成・設定・編集方法の詳細は [Grafana ドキュメント](https://docs.grafana.org/) を参照してください。

## クリーンアップ {#cleanup}

- 実行中の `kubectl port-forward` プロセスがあれば削除してください：

  {{< text bash >}}
  $ killall kubectl
  {{< /text >}}

- 他のタスクを試す予定がなければ、[Bookinfo のクリーンアップ](/zh/docs/examples/bookinfo/#cleanup)の手順に従い、アプリケーションを削除してください。
