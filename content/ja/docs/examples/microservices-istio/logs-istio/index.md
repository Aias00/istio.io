---
title: Istio の監視
overview: メッシュのメトリクスを収集・クエリする。
weight: 72

owner: istio/wg-docs-maintainers
test: no
---

監視は、マイクロサービスアーキテクチャへの移行を支える重要な要素です。

Istio では、デフォルトでマイクロサービス間トラフィックの監視機能が提供されています。
Istio ダッシュボードを使って、マイクロサービスをリアルタイムで監視できます。

Istio には、[Prometheus の時系列データベースと監視システム](https://prometheus.io)が組み込まれています。
Prometheus はさまざまなトラフィック関連メトリクスを収集し、[強力なクエリ言語](https://prometheus.io/docs/prometheus/latest/querying/basics/)を提供します。

以下は、Prometheus で Istio 関連のデータをクエリする例です。

1.  [http://my-istio-logs-database.io](http://my-istio-logs-database.io) から Prometheus UI にアクセスします（この `my-istio-logs-database.io` URL は[以前の設定](/ja/docs/examples/microservices-istio/bookinfo-kubernetes/#update-your-etc-hosts-configuration-file)で `/etc/hosts` に追加したものです）。

    {{< image width="80%" link="prometheus.png" caption="Prometheus Query UI" >}}

1.  **Expression** 入力欄で以下のクエリ例を実行します。**Execute** ボタンを押し、**Console** で結果を確認してください。この例では `tutorial` を名前空間としていますが、ご自身の名前空間に置き換えてください。
    リアルタイムトラフィックシミュレータを動かしていると、より良い結果が得られます。

        1. 名前空間内のすべてのリクエストをクエリ：

            {{< text plain >}}
            istio_requests_total{destination_service_namespace="tutorial", reporter="destination"}
            {{< /text >}}

        1. 名前空間内リクエストの合計：

            {{< text plain >}}
            sum(istio_requests_total{destination_service_namespace="tutorial", reporter="destination"})
            {{< /text >}}

        1. `reviews` マイクロサービスへのリクエスト：

            {{< text plain >}}
            istio_requests_total{destination_service_namespace="tutorial", reporter="destination",destination_service_name="reviews"}
            {{< /text >}}

        1. 過去 5 分間の `reviews` マイクロサービスインスタンスへの全リクエストの[リクエストレート](https://prometheus.io/docs/prometheus/latest/querying/functions/#rate)：

            {{< text plain >}}
            rate(istio_requests_total{destination_service_namespace="tutorial", reporter="destination",destination_service_name="reviews"}[5m])
            {{< /text >}}

上記のクエリで使われている `istio_requests_total` は、Istio 標準のメトリクスです。
他にも、特に Envoy（[Envoy](https://www.envoyproxy.io) は Istio の Sidecar プロキシ）に関するメトリクスも観察できます。**insert metric at cursor** ドロップダウンで収集されたデータを確認できます。

## 次のステップ {#next-steps}

チュートリアルの完了おめでとうございます！

これらの `demo` インストールタスクは、Istio をさらに学ぶための出発点です：

- [リクエストルーティングの設定](/ja/docs/tasks/traffic-management/request-routing/)
- [フォールトインジェクション](/ja/docs/tasks/traffic-management/fault-injection/)
- [トラフィックシフト](/ja/docs/tasks/traffic-management/traffic-shifting/)
- [Prometheus でメトリクスをクエリ](/ja/docs/tasks/observability/metrics/querying-metrics/)
- [Grafana でメトリクスを可視化](/ja/docs/tasks/observability/metrics/using-istio-dashboard/)
- [外部サービスへのアクセス](/ja/docs/tasks/traffic-management/egress/egress-control/)
- [ネットワークの可視化](/ja/docs/tasks/observability/kiali/)

Istio 製品をカスタマイズする前に、以下のリソースもご覧ください：

- [デプロイメントモデル](/ja/docs/ops/deployment/deployment-models/)
- [デプロイメントのベストプラクティス](/ja/docs/ops/best-practices/deployment/)
- [Pod と Service](/ja/docs/ops/deployment/application-requirements/)
- [インストール](/ja/docs/setup/)

## Istio コミュニティに参加しよう {#join-the-Istio-community}

ご意見・ご質問は、[Istio コミュニティ](/ja/get-involved/) への参加をお待ちしています。
