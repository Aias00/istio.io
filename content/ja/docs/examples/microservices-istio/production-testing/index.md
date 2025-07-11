---
title: 本番テスト
overview: 本番環境でマイクロサービスの新バージョンをテストする。
weight: 40
owner: istio/wg-docs-maintainers
test: no
---

{{< boilerplate work-in-progress >}}

本番環境でマイクロサービスをテストしましょう！

## 個別マイクロサービスのテスト {#testing-individual-microservices}

1. テスト Pod からサービスの 1 つに HTTP リクエストを送信します：

   {{< text bash >}}
   $ kubectl exec -it $(kubectl get pod -l app=curl -o jsonpath='{.items[0].metadata.name}') -- curl http://ratings:9080/ratings/7
   {{< /text >}}

## カオステスト {#chaos-testing}

本番環境で[カオステスト](http://www.boyter.org/2016/07/chaos-testing-engineering/)を実施し、アプリケーションがどのように反応するかを確認しましょう。各カオス操作の後、アプリケーションの Web ページを確認し、何か変化があるか観察します。
`kubectl get pods` で Pod の状態も確認してください。

1. `details` サービスの 1 つの Pod でプロセスを強制終了します。

   {{< text bash >}}
   $ kubectl exec -it $(kubectl get pods -l app=details -o jsonpath='{.items[0].metadata.name}') -- pkill ruby
   {{< /text >}}

1. Pod の状態を確認します：

   {{< text bash >}}
   $ kubectl get pods
   NAME READY STATUS RESTARTS AGE
   details-v1-6d86fd9949-fr59p 1/1 Running 1 47m
   details-v1-6d86fd9949-mksv7 1/1 Running 0 47m
   details-v1-6d86fd9949-q8rrf 1/1 Running 0 48m
   productpage-v1-c9965499-hwhcn 1/1 Running 0 47m
   productpage-v1-c9965499-nccwq 1/1 Running 0 47m
   productpage-v1-c9965499-tjdjx 1/1 Running 0 48m
   ratings-v1-7bf577cb77-cbdsg 1/1 Running 0 47m
   ratings-v1-7bf577cb77-cz6jm 1/1 Running 0 47m
   ratings-v1-7bf577cb77-pq9kg 1/1 Running 0 48m
   reviews-v1-77c65dc5c6-5wt8g 1/1 Running 0 47m
   reviews-v1-77c65dc5c6-kjvxs 1/1 Running 0 48m
   reviews-v1-77c65dc5c6-r55tl 1/1 Running 0 47m
   curl-88ddbcfdd-l9zq4 1/1 Running 0 47m
   {{< /text >}}

   最初の Pod が 1 回再起動していることに注目してください。

1. `details` のすべての Pod でプロセスを強制終了します：

   {{< text bash >}}
   $ for pod in $(kubectl get pods -l app=details -o jsonpath='{.items[*].metadata.name}'); do echo terminating $pod; kubectl exec -it $pod -- pkill ruby; done
   {{< /text >}}

1. アプリケーションのページを確認します：

   {{< image width="80%"
       link="bookinfo-details-unavailable.png"
       caption="Bookinfo Web Application、詳細が利用できません"
       >}}

   詳細部分にエラーが表示され、本の詳細情報が表示されないことに注目してください。

1. Pod の状態を確認します：

   {{< text bash >}}
   $ kubectl get pods
   NAME READY STATUS RESTARTS AGE
   details-v1-6d86fd9949-fr59p 1/1 Running 2 48m
   details-v1-6d86fd9949-mksv7 1/1 Running 1 48m
   details-v1-6d86fd9949-q8rrf 1/1 Running 1 49m
   productpage-v1-c9965499-hwhcn 1/1 Running 0 48m
   productpage-v1-c9965499-nccwq 1/1 Running 0 48m
   productpage-v1-c9965499-tjdjx 1/1 Running 0 48m
   ratings-v1-7bf577cb77-cbdsg 1/1 Running 0 48m
   ratings-v1-7bf577cb77-cz6jm 1/1 Running 0 48m
   ratings-v1-7bf577cb77-pq9kg 1/1 Running 0 49m
   reviews-v1-77c65dc5c6-5wt8g 1/1 Running 0 48m
   reviews-v1-77c65dc5c6-kjvxs 1/1 Running 0 49m
   reviews-v1-77c65dc5c6-r55tl 1/1 Running 0 48m
   curl-88ddbcfdd-l9zq4 1/1 Running 0 48m
   {{< /text >}}

   最初の Pod は 2 回、他の 2 つの `details` Pod は 1 回再起動しています。
   Pod が `Running` 状態になるまで `Error` や `CrashLoopBackOff` 状態になることもあります。

1. 無限ループでトラフィックをシミュレートしている場合は、Ctrl-C で停止してください。

これらのケースでも、アプリケーションはクラッシュしませんでした。
`details` マイクロサービスのクラッシュは、他のマイクロサービスの障害にはつながりません。
この挙動は**カスケード障害**が発生しないことを示しています。
代わりに、サービスは**段階的に劣化**します。1 つのマイクロサービスがクラッシュしても、アプリケーションは有用な機能を提供し続けます。
書籍のレビューや基本情報は表示されます。

これで[レビューアプリケーションの新バージョン追加](/ja/docs/examples/microservices-istio/add-new-microservice-version)の準備ができました。
