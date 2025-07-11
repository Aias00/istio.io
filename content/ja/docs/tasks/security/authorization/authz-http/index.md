---
title: HTTP トラフィック
description: HTTP トラフィックのアクセス制御の設定方法を紹介します。
weight: 10
keywords: [security, access-control, rbac, authorization]
aliases:
  - /zh/docs/tasks/security/role-based-access-control.html
  - /zh/docs/tasks/security/authz-http/
owner: istio/wg-security-maintainers
test: yes
---

このタスクでは、Istio メッシュ内の HTTP トラフィックに対して `ALLOW` アクションの Istio 認可ポリシーを設定する方法を紹介します。

## 始める前に {#before-you-begin}

このタスクを始める前に、以下を実施してください：

- [Istio 認可の概念](/ja/docs/concepts/security/#authorization)を読んでください。

- [Istio インストールガイド](/ja/docs/setup/install/istioctl/)に従って Istio をインストールし、双方向 TLS を有効にしてください。

- [Bookinfo](/ja/docs/examples/bookinfo/#deploying-the-application) サンプルアプリケーションをデプロイしてください。

Bookinfo アプリをデプロイした後、`http://$GATEWAY_URL/productpage` で product ページにアクセスすると、次の内容が表示されます：

- **Book Details** はページ中央にあり、書籍の種類、ページ数、出版社などが含まれます。
- **Book Reviews** はページ下部に表示されます。

ページをリロードすると、product ページのレビュー部分に赤い星、黒い星、または星なしの異なるバージョンが順番に表示されます。

{{< tip >}}
ブラウザで期待される出力が表示されない場合は、数秒後に再試行してください。キャッシュやその他の伝送オーバーヘッドにより遅延が発生することがあります。
{{< /tip >}}

{{< warning >}}
このタスクでは、以下の例でポリシーのプリンシパルやネームスペースを使用するため、双方向 TLS の有効化が必要です。
{{< /warning >}}

## HTTP トラフィックのワークロードにアクセス制御を設定する {#configure-access-control-for-workloads-using-http-traffic}

Istio を使うと、メッシュ内の {{< gloss "workload" >}}ワークロード{{< /gloss >}} に簡単にアクセス制御を設定できます。
このタスクでは、Istio 認可を使ったアクセス制御の設定方法を紹介します。まず、シンプルな `allow-nothing` ポリシーを設定して、ワークロードへのすべてのリクエストを拒否し、その後、段階的にアクセス権を拡張していきます。

1. 次のコマンドで `default` 名前空間に `allow-nothing` ポリシーを作成します。
   このポリシーは `selector` フィールドがなく、`default` 名前空間内のすべてのワークロードに適用されます。
   `spec:` フィールドが空 `{}` なので、すべてのトラフィックが許可されず、すべてのリクエストが拒否されます。

   {{< text bash >}}
   $ kubectl apply -f - <<EOF
   apiVersion: security.istio.io/v1
   kind: AuthorizationPolicy
   metadata:
   name: allow-nothing
   namespace: default
   spec:
   {}
   EOF
   {{< /text >}}

   ブラウザで Bookinfo の `productpage` (`http://$GATEWAY_URL/productpage`) ページにアクセスします。
   "RBAC: access denied" というエラーが表示されます。これは `deny-all` ポリシーが期待通りに動作し、
   Istio によるワークロードへのアクセスがすべて拒否されていることを示します。

1. 次のコマンドで `productpage-viewer` ポリシーを作成し、`productpage` ワークロードへの `GET` メソッドによるアクセスを許可します。
   このポリシーの `rules` には `from` フィールドがないため、すべてのリクエスト元（ユーザーやワークロード）が許可されます：

   {{< text bash >}}
   $ kubectl apply -f - <<EOF
   apiVersion: security.istio.io/v1
   kind: AuthorizationPolicy
   metadata:
   name: "productpage-viewer"
   namespace: default
   spec:
   selector:
   matchLabels:
   app: productpage
   action: ALLOW
   rules:

   - to: - operation:
     methods: ["GET"]
     EOF
     {{< /text >}}

   ブラウザで Bookinfo の `productpage` (`http://$GATEWAY_URL/productpage`) にアクセスします。
   "Bookinfo Sample" ページが表示されますが、次のようなエラーが表示されます：

   - `Error fetching product details`
   - `Error fetching product reviews`

   これらのエラーは想定通りです。`productpage` ワークロードが `details` や `reviews` ワークロードへのアクセス権を持っていないためです。
   次に、他のワークロードへのアクセスを許可するポリシーを設定します。

1. 次のコマンドで `details-viewer` ポリシーを作成し、`productpage` ワークロードが `GET` メソッドで
   `cluster.local/ns/default/sa/bookinfo-productpage` ServiceAccount を使って `details` ワークロードへアクセスできるようにします：

   {{< text bash >}}
   $ kubectl apply -f - <<EOF
   apiVersion: security.istio.io/v1
   kind: AuthorizationPolicy
   metadata:
   name: "details-viewer"
   namespace: default
   spec:
   selector:
   matchLabels:
   app: details
   action: ALLOW
   rules:

   - from: - source:
     principals: ["cluster.local/ns/default/sa/bookinfo-productpage"]
     to: - operation:
     methods: ["GET"]
     EOF
     {{< /text >}}

1. 次のコマンドで `reviews-viewer` ポリシーを作成し、`productpage` ワークロードが `GET` メソッドで
   `cluster.local/ns/default/sa/bookinfo-productpage` ServiceAccount を使って `reviews` ワークロードへアクセスできるようにします：

   {{< text bash >}}
   $ kubectl apply -f - <<EOF
   apiVersion: security.istio.io/v1
   kind: AuthorizationPolicy
   metadata:
   name: "reviews-viewer"
   namespace: default
   spec:
   selector:
   matchLabels:
   app: reviews
   action: ALLOW
   rules:

   - from: - source:
     principals: ["cluster.local/ns/default/sa/bookinfo-productpage"]
     to: - operation:
     methods: ["GET"]
     EOF
     {{< /text >}}

   ブラウザで Bookinfo の `productpage` (`http://$GATEWAY_URL/productpage`) にアクセスします。
   "Bookinfo Sample" ページの左下に "Book Details"、右下に "Book Reviews" が表示されます。
   ただし "Book Reviews" 部分には `Ratings service currently unavailable` というエラーが表示されます。

   これは、`reviews` ワークロードが `ratings` ワークロードへのアクセス権を持っていないためです。
   この問題を解決するには、`reviews` ワークロードが `ratings` ワークロードへアクセスできるようにポリシーを追加します。

1. 次のコマンドで `ratings-viewer` ポリシーを作成し、`reviews` ワークロードが `GET` メソッドで
   `cluster.local/ns/default/sa/bookinfo-reviews` ServiceAccount を使って `ratings` ワークロードへアクセスできるようにします：

   {{< text bash >}}
   $ kubectl apply -f - <<EOF
   apiVersion: security.istio.io/v1
   kind: AuthorizationPolicy
   metadata:
   name: "ratings-viewer"
   namespace: default
   spec:
   selector:
   matchLabels:
   app: ratings
   action: ALLOW
   rules:

   - from: - source:
     principals: ["cluster.local/ns/default/sa/bookinfo-reviews"]
     to: - operation:
     methods: ["GET"]
     EOF
     {{< /text >}}

   ブラウザで Bookinfo の `productpage` (`http://$GATEWAY_URL/productpage`) にアクセスします。
   "Book Reviews" 部分に「黒い星」と「赤い星」の評価が表示されます。

   **おめでとうございます！** HTTP トラフィックを利用するワークロードに対して認可ポリシーによるアクセス制御を適用できました。

## クリーンアップ {#clean-up}

設定からすべての認可ポリシーを削除します：

{{< text bash >}}
$ kubectl delete authorizationpolicy.security.istio.io/allow-nothing
$ kubectl delete authorizationpolicy.security.istio.io/productpage-viewer
$ kubectl delete authorizationpolicy.security.istio.io/details-viewer
$ kubectl delete authorizationpolicy.security.istio.io/reviews-viewer
$ kubectl delete authorizationpolicy.security.istio.io/ratings-viewer
{{< /text >}}
