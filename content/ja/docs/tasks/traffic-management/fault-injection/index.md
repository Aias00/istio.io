---
title: フォールトインジェクション
description: このタスクでは、障害を注入してアプリケーションのレジリエンスをテストする方法を説明します。
weight: 20
keywords: [traffic-management, fault-injection]
aliases:
  - /zh/docs/tasks/fault-injection.html
owner: istio/wg-networking-maintainers
test: yes
---

このタスクでは、障害（フォールト）を注入してアプリケーションのレジリエンス（耐障害性）をテストする方法を説明します。

## 始める前に {#before-you-begin}

- [インストールガイド](/ja/docs/setup/) の手順に従って Istio を構成してください。

- サンプルアプリケーション [Bookinfo](/ja/docs/examples/bookinfo/) をデプロイし、[デフォルトのデスティネーションルール](/ja/docs/examples/bookinfo/#apply-default-destination-rules) を適用してください。

- [トラフィック管理](/ja/docs/concepts/traffic-management) の概念ドキュメントでフォールトインジェクションの解説を確認してください。

- [リクエストルーティングの構成](/ja/docs/tasks/traffic-management/request-routing/)タスクを実行するか、以下のコマンドでアプリケーションのバージョンルーティングを初期化してください：

  {{< text bash >}}
    $ kubectl apply -f @samples/bookinfo/networking/virtual-service-all-v1.yaml@
    $ kubectl apply -f @samples/bookinfo/networking/virtual-service-reviews-test-v2.yaml@
  {{< /text >}}

- 上記の設定後、リクエストの流れは次の通りです：
  - `productpage` → `reviews:v2` → `ratings`（ユーザー `jason` の場合）
  - `productpage` → `reviews:v1`（その他のユーザー）

## HTTP 遅延フォールトの注入 {#injecting-an-http-delay-fault}

マイクロサービスアプリケーション Bookinfo のレジリエンスをテストするため、ユーザー `jason` に対して `reviews:v2` と `ratings` サービス間に 7 秒の遅延を注入します。
このテストでは、Bookinfo アプリケーションに意図的に仕込まれたバグを発見できます。

`reviews:v2` サービスが `ratings` サービスを呼び出す際、10 秒のハードコードされた接続タイムアウトがあります。
そのため、7 秒の遅延を注入しても、エンドツーエンドの処理はエラーなく完了するはずです。

1. テストユーザー `jason` のトラフィックに遅延を注入するフォールトインジェクションルールを作成します：

   {{< text bash >}}
    $ kubectl apply -f @samples/bookinfo/networking/virtual-service-ratings-test-delay.yaml@
   {{< /text >}}

1. ルールが作成されたことを確認します：

   {{< text bash yaml >}}
    $ kubectl get virtualservice ratings -o yaml
    apiVersion: networking.istio.io/v1
    kind: VirtualService
    spec:
    hosts:

    - ratings
      http:
    - fault:
      delay:
      fixedDelay: 7s
      percentage:
      value: 100
      match:
      - headers:
        end-user:
        exact: jason
        route:
      - destination:
        host: ratings
        subset: v1
    - route: - destination:
      host: ratings
      subset: v1
      {{< /text >}}

    新しいルールが全 Pod に伝播するまで数秒かかる場合があります。

## 遅延設定のテスト {#testing-the-delay-configuration}

1. ブラウザで [Bookinfo](/ja/docs/examples/bookinfo) アプリを開きます。

1. ユーザー `jason` で `/productpage` ページにログインします。

   Bookinfo のホームページは約 7 秒で読み込まれ、エラーは発生しないはずです。
   しかし、Reviews セクションにエラーメッセージが表示されます：

   {{< text plain >}}
    Sorry, product reviews are currently unavailable for this book.
   {{< /text >}}

1. ページの応答時間を確認します：

   1. ブラウザの「開発者ツール」メニューを開く
   1. 「ネットワーク」タブを開く
   1. `/productpage` ページをリロードし、ページの読み込みに約 6 秒かかっていることを確認します。

## 原因の理解 {#understanding-what-happened}

ハードコードされたタイムアウトにより、`reviews` サービスが失敗するというバグが見つかりました。

想定通り、7 秒の遅延を注入しても `reviews` サービスには影響しないはずでした（`reviews` と `ratings` の間のタイムアウトは 10 秒）。
しかし、`productpage` と `reviews` の間にも 3 秒のハードコードタイムアウトと 1 回のリトライ（合計 6 秒）があり、`productpage` から `reviews` への呼び出しが 6 秒でタイムアウトしてエラーとなりました。

この種のバグは、異なるチームが独立してマイクロサービスを開発する企業アプリケーションでよく発生します。
Istio のフォールトインジェクションルールは、最終ユーザーに影響を与えずにこのような問題を特定するのに役立ちます。

{{< tip >}}
このフォールトインジェクションはユーザー `jason` のみに影響します。他のユーザーでログインした場合、遅延は発生しません。
{{< /tip >}}

## バグの修正 {#fixing-the-bug}

この問題の一般的な解決策は次の通りです：

1. `productpage` と `reviews` の間のタイムアウトを延長する、または `reviews` と `ratings` のタイムアウトを短縮する
1. 修正したマイクロサービスを終了し再起動する
1. `/productpage` ページが正常に応答し、エラーがないことを確認する

ただし、`reviews` サービスの v3 バージョンではこの問題が修正されています。
`reviews:v3` サービスでは、`reviews` と `ratings` のタイムアウトが 10 秒から 2.5 秒に短縮されており、下流の `productpage` リクエストのタイムアウト（3 秒）よりも短くなっています。

[トラフィックシフト](/ja/docs/tasks/traffic-management/traffic-shifting/)タスクの手順で全トラフィックを `reviews:v3` に移した場合、遅延ルールを 2 秒など 2.5 秒未満に変更し、エンドツーエンドでエラーが発生しないことを確認できます。

## HTTP abort フォールトの注入 {#injecting-an-http-abort-fault}

マイクロサービスのレジリエンスをテストするもう一つの方法は、HTTP abort フォールトを注入することです。
このタスクでは、テストユーザー `jason` に対して `ratings` マイクロサービスに HTTP abort を注入します。

この場合、ページは即座に読み込まれ、「Ratings service is currently unavailable」のようなメッセージが表示されることを期待します。

1. ユーザー `jason` に HTTP abort を送信するフォールトインジェクションルールを作成します：

   {{< text bash >}}
    $ kubectl apply -f @samples/bookinfo/networking/virtual-service-ratings-test-abort.yaml@
   {{< /text >}}

1. ルールが作成されたことを確認します：

   {{< text bash yaml >}}
    $ kubectl get virtualservice ratings -o yaml
    apiVersion: networking.istio.io/v1
    kind: VirtualService
    spec:
    hosts:

    - ratings
      http:
    - fault:
      abort:
      httpStatus: 500
      percentage:
      value: 100
      match:
      - headers:
        end-user:
        exact: jason
        route:
      - destination:
        host: ratings
        subset: v1
    - route: - destination:
      host: ratings
      subset: v1
     {{< /text >}}

## abort 設定のテスト {#testing-the-abort-configuration}

1. ブラウザで [Bookinfo](/ja/docs/examples/bookinfo) アプリを開きます。

1. ユーザー `jason` で `/productpage` ページにログインします。

   ルールが全 Pod に伝播していれば、ページは即座に読み込まれ、「Ratings service is currently unavailable」というメッセージが表示されるはずです。

1. ユーザー `jason` でログアウトするか、シークレットウィンドウ（または別のブラウザ）で Bookinfo アプリを開くと、`/productpage` では `reviews:v1`（`ratings` には全くアクセスしない）が呼ばれるため、エラーメッセージは表示されません。

## クリーンアップ {#cleanup}

1. アプリケーションのルーティングルールを削除します：

   {{< text bash >}}
    $ kubectl delete -f @samples/bookinfo/networking/virtual-service-all-v1.yaml@
   {{< /text >}}

1. 追加のタスクを行わない場合は、[Bookinfo のクリーンアップ](/ja/docs/examples/bookinfo/#cleanup)の手順でアプリケーションを削除してください。
