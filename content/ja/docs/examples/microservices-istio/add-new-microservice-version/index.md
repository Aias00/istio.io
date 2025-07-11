---
title: reviews の新バージョンを追加する
overview: 新しいバージョンのマイクロサービスをデプロイします。
weight: 50
owner: istio/wg-docs-maintainers
test: no
---

このモジュールでは、レビュアーが提供した評価の星と色を返す reviews サービスの新バージョン v2 をデプロイします。
実際のシナリオでは、まず模擬環境で静的解析テスト、ユニットテスト、統合テスト、エンドツーエンドテスト、検証を行ってからデプロイします。

1. `app=reviews` ラベルのない新バージョンの `reviews` マイクロサービスをデプロイします。このラベルがなければ、新バージョンの `reviews` はサービスに選ばれず、プロダクションコードからも呼ばれません。次のコマンドで v2 の `reviews` マイクロサービスをデプロイし、ラベル `app=reviews` を `app=reviews_test` に置き換えます：

   {{< text bash >}}
   $ curl -s {{< github_file >}}/samples/bookinfo/platform/kube/bookinfo.yaml | sed 's/app: reviews/app: reviews_test/' | kubectl apply -l app=reviews_test,version=v2 -f -
   deployment.apps/reviews-v2 created
   {{< /text >}}

1. アプリにアクセスし、デプロイしたマイクロサービスがアプリを壊していないことを確認します。

1. 以前にデプロイしたテストコンテナを使って、クラスター内部から新バージョンのマイクロサービスをテストします。新バージョンはテスト中に `ratings` マイクロサービスのプロダクション Pod にアクセスすることに注意してください。また、新バージョンはまだ `reviews` サービスに選ばれていないため、Pod IP を使ってアクセスする必要があります。

   1. Pod IP を取得します：

      {{< text bash >}}
      $ REVIEWS_V2_POD_IP=$(kubectl get pod -l app=reviews_test,version=v2 -o jsonpath='{.items[0].status.podIP}')
      $ echo $REVIEWS_V2_POD_IP
      {{< /text >}}

   1. Pod にリクエストを送り、正しい結果が返るか確認します：

      {{< text bash >}}
      $ kubectl exec $(kubectl get pod -l app=curl -o jsonpath='{.items[0].metadata.name}') -- curl -sS "$REVIEWS_V2_POD_IP:9080/reviews/7"
      {"id": "7","reviews": [{ "reviewer": "Reviewer1", "text": "An extremely entertaining play by Shakespeare. The slapstick humour is refreshing!", "rating": {"stars": 5, "color": "black"}},{ "reviewer": "Reviewer2", "text": "Absolutely fun and entertaining. The play lacks thematic depth when compared to other plays by Shakespeare.", "rating": {"stars": 4, "color": "black"}}]}
      {{< /text >}}

   1. 10 回連続でリクエストを送り、簡易負荷テストを行います：

      {{< text bash >}}
      $ kubectl exec $(kubectl get pod -l app=curl -o jsonpath='{.items[0].metadata.name}') -- sh -c "for i in 1 2 3 4 5 6 7 8 9 10; do curl -o /dev/null -s -w '%{http_code}\n' $REVIEWS_V2_POD_IP:9080/reviews/7; done"
      200
      200
      ...
      {{< /text >}}

1. これまでの手順で新バージョンの `reviews` が正常に動作することを確認できたので、デプロイを進めます。まず、単一レプリカのサービスを本番にデプロイし、実際のプロダクショントラフィックが新バージョンに流れ始めます。現状では、75% のトラフィックが旧バージョン（3 つの Pod）、25% が新バージョン（1 つの Pod）に流れます。

   **reviews v2** をデプロイするには、新バージョンに `app=reviews` ラベルを付与し、`reviews` サービスでアドレス指定できるようにします。

   {{< text bash >}}
   $ kubectl label pods -l version=v2 app=reviews --overwrite
   pod "reviews-v2-79c8c8c7c5-4p4mn" labeled
   {{< /text >}}

1. アプリページにアクセスし、評価に黒い星が表示されることを確認します。何度かページをリロードすると、約 25% の確率で星が表示され、約 75% では表示されません。

   {{< image width="80%"
       link="bookinfo-reviews-v2.png"
       caption="黒い星で評価された Bookinfo Web アプリ"
       >}}

1. 実際の運用で新バージョンに問題が発生した場合は、すぐに新バージョンのデプロイを削除できます。これで旧バージョンのみが利用されます：

   {{< text bash >}}
   $ kubectl delete deployment reviews-v2
   $ kubectl delete pod -l app=reviews,version=v2
   deployment.apps "reviews-v2" deleted
   pod "reviews-v2-79c8c8c7c5-4p4mn" deleted
   {{< /text >}}

   設定変更が反映されるまで少し待ち、アプリページを何度かリロードすると、黒い星が表示されなくなります。

   新バージョンを復元するには：

   {{< text bash >}}
   $ kubectl apply -l app=reviews,version=v2 -f {{< github_file >}}/samples/bookinfo/platform/kube/bookinfo.yaml
   deployment.apps/reviews-v2 created
   {{< /text >}}

   アプリページを何度かリロードすると、約 25% の確率で黒い星が表示されます。

1. 次に、新バージョンのレプリカ数を増やします。段階的にレプリカ数を増やし、エラーが増えていないか慎重に確認します。

   {{< text bash >}}
   $ kubectl scale deployment reviews-v2 --replicas=3
   deployment.apps/reviews-v2 scaled
   {{< /text >}}

   これでアプリページを何度かリロードすると、黒い星が表示される確率が約半分になります。

1. 旧バージョンを停止します：

   {{< text bash >}}
   $ kubectl delete deployment reviews-v1
   deployment.apps "reviews-v1" deleted
   {{< /text >}}

   以降、アプリページにアクセスすると、黒い星付きの `reviews` のみが返されます。

これらの手順で `reviews` のローリングアップデートを実施しました。まず新バージョンをデプロイし、プロダクショントラフィックを流さずにテストしました。テストで正しい結果が得られることを確認し、本番リリース後は段階的にトラフィックを増やし、最終的に旧バージョンを停止しました。

ここからは、以下のサンプルタスクでデプロイ戦略をさらに改善できます。まず、本番環境で新バージョンのエンドツーエンドテストを行います。これは、リクエストパラメータ（例：Cookie に保存されたユーザー名）を使ってトラフィックを新バージョンに振り分けることで実現できます。次に、新バージョンの本番トラフィックをミラーリングし、誤った結果やエラーが発生しないか確認します。最後に、リリース時のトラフィック制御をより細かく行います。たとえば、最初は 1% だけリリースし、問題がなければ 1 時間ごとに 1% ずつ増やすなどです。Istio はこれらのタスクを直接支援し、Kubernetes の価値を高めます。デプロイの詳細やベストプラクティスは[デプロイモデル](/ja/docs/ops/deployment/deployment-models/)を参照してください。

ここで 2 つの選択肢があります：

1. **サービスメッシュ**を使う。サービスメッシュでは、すべてのレポート、ルーティング、ポリシー、セキュリティロジックを**Sidecar** プロキシに集約し、アプリ Pod に**透過的に**注入します。ビジネスロジックはアプリのコードに残り、アプリのコードを変更する必要はありません。

1. 必要な機能をアプリケーションコードで実装する。多くの機能はさまざまなライブラリで提供されています（例：Java 用の Netflix [Hystrix](https://github.com/Netflix/Hystrix) など）。ただし、これらのライブラリを使うにはコードの修正が必要です。ビジネスロジックとレポート、ルーティング、ポリシー、ネットワークロジックが混在し、コード量も増加します。マイクロサービスごとに異なる言語を使っている場合、複数のライブラリを学び、使い、更新する必要があります。

[Istio サービスメッシュ](/ja/about/service-mesh/)を参照すると、本記事や他のページで紹介したタスクを Istio がどのように実現するかが分かります。

次のモジュールでは、Istio の機能を探索します。

[`productpage` で Istio を有効化](/ja/docs/examples/microservices-istio/add-istio/)する準備ができました。
