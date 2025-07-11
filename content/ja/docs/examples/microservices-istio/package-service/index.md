---
title: Docker で ratings サービスを実行する
overview: Docker コンテナ内でマイクロサービスを実行します。

weight: 20

owner: istio/wg-docs-maintainers
test: no
---

{{< boilerplate work-in-progress >}}

このモジュールでは、[Docker](https://www.docker.com) イメージを作成し、ローカルで実行する方法を紹介します。

1. マイクロサービス `ratings` の [`Dockerfile`](https://docs.docker.com/engine/reference/builder/) をダウンロードします。

   {{< text bash >}}
   $ curl -s {{< github_file >}}/samples/bookinfo/src/ratings/Dockerfile -o Dockerfile
   {{< /text >}}

1. この `Dockerfile` を確認します。

   {{< text bash >}}
   $ cat Dockerfile
   {{< /text >}}

   ファイルがコンテナのファイルシステムにコピーされ、前のモジュールで実行した `npm install` コマンドが実行されていることに注目してください。
   `CMD` コマンドは Docker に `ratings` サービスをポート `9080` で実行するよう指示しています。

1. 環境変数にユーザー ID を保存します。このユーザー ID は docker イメージのタグ付けに使います。
   例: `user`。

   {{< text bash >}}
   $ export USER=user
   {{< /text >}}

1. `Dockerfile` からイメージをビルドします：

   {{< text bash >}}
   $ docker build -t $USER/ratings .
   ...
   Step 9/9 : CMD node /opt/microservices/ratings.js 9080
   ---> Using cache
   ---> 77c6a304476c
   Successfully built 77c6a304476c
   Successfully tagged user/ratings:latest
   {{< /text >}}

1. Docker で `ratings` サービスを実行します。次の [docker run](https://docs.docker.com/engine/reference/commandline/run/) コマンドは、コンテナの `9080` ポートをマシンの `9081` ポートに公開し、`9081` ポートで `ratings` マイクロサービスにアクセスできるようにします。

   {{< text bash >}}
   $ docker run --name my-ratings --rm -d -p 9081:9080 $USER/ratings
   {{< /text >}}

1. ブラウザで [http://localhost:9081/ratings/7](http://localhost:9081/ratings/7) にアクセスするか、以下の `curl` コマンドを使います：

   {{< text bash >}}
   $ curl localhost:9081/ratings/7
   {"id":7,"ratings":{"Reviewer1":5,"Reviewer2":4}}
   {{< /text >}}

1. 実行中のコンテナを確認します。[docker ps](https://docs.docker.com/engine/reference/commandline/ps/) コマンドで、すべての実行中コンテナをリストし、イメージが `<your user name>/ratings` であることを確認します。

   {{< text bash >}}
   $ docker ps
   CONTAINER ID IMAGE COMMAND CREATED STATUS PORTS NAMES
   47e8c1fe6eca user/ratings "docker-entrypoint.s…" 2 minutes ago Up 2 minutes 0.0.0.0:9081->9080/tcp elated_stonebraker
   ...
   {{< /text >}}

1. 実行中のコンテナを停止します：

   {{< text bash >}}
   $ docker stop my-ratings
   {{< /text >}}

これで、単一サービスをコンテナ化する方法が分かりました。次は[アプリケーションを Kubernetes クラスタにデプロイする](/ja/docs/examples/microservices-istio/bookinfo-kubernetes)方法を学びましょう。
