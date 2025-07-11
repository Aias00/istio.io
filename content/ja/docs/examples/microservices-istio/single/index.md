---
title: ローカルでマイクロサービスを実行する
overview: ローカルマシンで単一サービスの作業方法を学びます。
weight: 10
owner: istio/wg-docs-maintainers
test: no
---

{{< boilerplate work-in-progress >}}

マイクロサービスアーキテクチャが登場する前は、開発チームはアプリケーション全体を巨大なソフトウェアとして構築・デプロイ・実行していました。モジュールの小さな変更をテストする場合でも、開発者は単体テストだけでなく、アプリ全体を再ビルドする必要がありました。そのためビルドには多くの時間がかかりました。ビルドが終わると、開発者はアプリケーションのバージョンをテストサーバーにデプロイします。サービスはリモートまたはローカルマシンで実行されます。後者の場合、開発者はローカルマシンにかなり複雑な環境をインストール・管理していました。

マイクロサービスアーキテクチャ時代では、開発者は小さなソフトウェアサービスを作成・ビルド・テスト・実行します。ビルドは高速です。[Node.js](https://nodejs.org/ja/) のようなモダンなフレームワークを使えば、サービスは通常のプロセスとして動作するため、テストのために複雑なサービス環境をインストール・管理する必要はありません。サービスをテストするためだけにどこかの環境にデプロイする必要はなく、ローカルマシンでビルドしてそのまま実行できます。

このモジュールでは、ローカルマシンで単一サービスを開発する際のさまざまな側面を扱います。ただし、コードを書く必要はなく、既存の `ratings` サービスをビルド・実行・テストするだけです。

`ratings` サービスは [Node.js](https://nodejs.org/ja/) で書かれた、単独で動作する小さな Web アプリケーションです。他の Web アプリケーションと同様の動作をします：

- 引数で受け取ったポートでリッスンする
- `/ratings/{productID}` パスへの `HTTP GET` リクエストを受け付け、クライアントが指定した `productID` に一致する商品の評価を返す
- `/ratings/{productID}` パスへの `HTTP POST` リクエストを受け付け、指定した `productID` に一致する商品の評価を更新する

以下の手順でアプリケーションのコードをダウンロードし、依存関係をインストールし、ローカルで実行します：

1. [サービスコード]({{< github_blob >}}/samples/bookinfo/src/ratings/ratings.js)と
   [package ファイル]({{< github_blob >}}/samples/bookinfo/src/ratings/package.json)を
   別ディレクトリにダウンロードします：

   {{< text bash >}}
   $ mkdir ratings
   $ cd ratings
   $ curl -s {{< github_file >}}/samples/bookinfo/src/ratings/ratings.js -o ratings.js
   $ curl -s {{< github_file >}}/samples/bookinfo/src/ratings/package.json -o package.json
   {{< /text >}}

1. サービスのコードを確認し、以下の点に注目してください：

   - Web サーバーの特徴：
     - ポートでリッスン
     - リクエストとレスポンスの処理
   - HTTP 関連の側面：
     - リクエストヘッダー
     - パス
     - ステータスコード

   {{< tip >}}
   Node.js では、Web サーバーの機能はアプリケーションコードに組み込まれています。
   Node.js Web アプリケーションは独立したプロセスとして動作します。
   {{< /tip >}}

1. Node.js アプリケーションは JavaScript で書かれているため、明示的なビルドステップはありません。
   代わりに[ジャストインタイムコンパイル](https://ja.wikipedia.org/wiki/Just-in-time%E3%82%B3%E3%83%B3%E3%83%91%E3%82%A4%E3%83%AB)が使われます。
   Node.js アプリケーションのビルドは、依存ライブラリのインストールを意味します。
   `ratings` サービスの依存ライブラリを、サービスコードと package ファイルを保存したディレクトリでインストールします：

   {{< text bash >}}
   $ npm install
   npm notice created a lockfile as package-lock.json. You should commit this file.
   npm WARN ratings No description
   npm WARN ratings No repository field.
   npm WARN ratings No license field.

   added 24 packages in 2.094s
   {{< /text >}}

1. `9080` を引数に渡してサービスを実行します。アプリケーションは 9080 ポートでリッスンします。

   {{< text bash >}}
   $ npm start 9080

   > @ start /tmp/ratings
   > node ratings.js "9080"
   > Server listening on: http://0.0.0.0:9080 > {{< /text >}}

{{< tip >}}
この `ratings` サービスは Web アプリケーションなので、他の Web アプリケーションと同様にアクセスできます。ブラウザや、[`curl`](https://curl.haxx.se) や [`Wget`](https://www.gnu.org/software/wget/) のようなコマンドライン Web クライアントを使えます。ローカルで `ratings` サービスを実行しているので、`localhost` ホスト名でアクセスできます。
{{< /tip >}}

1. ブラウザで [http://localhost:9080/ratings/7](http://localhost:9080/ratings/7) を開くか、`curl` コマンドで `ratings` にアクセスします：

   {{< text bash >}}
   $ curl localhost:9080/ratings/7
   {"id":7,"ratings":{"Reviewer1":5,"Reviewer2":4}}
   {{< /text >}}

1. `curl` コマンドの POST メソッドで商品の評価を 1 に設定します：

   {{< text bash >}}
   $ curl -X POST localhost:9080/ratings/7 -d '{"Reviewer1":1,"Reviewer2":1}'
   {"id":7,"ratings":{"Reviewer1":1,"Reviewer2":1}}
   {{< /text >}}

1. 更新された評価を確認します：

   {{< text bash >}}
   $ curl localhost:9080/ratings/7
   {"id":7,"ratings":{"Reviewer1":1,"Reviewer2":1}}
   {{< /text >}}

1. サービスを実行しているターミナルで `Ctrl-C` を押してサービスを停止します。

おめでとうございます。これでローカルマシンでサービスをビルド・テスト・実行できるようになりました！

これで[サービスをコンテナ化する方法](/ja/docs/examples/microservices-istio/package-service)の準備ができました。
