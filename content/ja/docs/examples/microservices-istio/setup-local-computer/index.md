---
title: ローカルマシンのセットアップ
overview: このチュートリアル用にローカルマシンをセットアップします。
weight: 3
owner: istio/wg-docs-maintainers
test: no
---

{{< boilerplate work-in-progress >}}

このモジュールでは、チュートリアル用にローカルマシンを準備します。

1. [`curl`](https://curl.haxx.se/download.html) をインストールします。

1. [Node.js](https://nodejs.org/en/download/) をインストールします。

1. [Docker](https://docs.docker.com/install/) をインストールします。

1. [`kubectl`](https://kubernetes.io/ja/docs/tasks/tools/#kubectl) をインストールします。

1. チュートリアルで受け取った、または前のモジュールで自分で作成した設定ファイルを `KUBECONFIG` 環境変数に設定します。

   {{< text bash >}}
   $ export KUBECONFIG=<前のモジュールで受け取った、または作成したファイル>
   {{< /text >}}

1. 現在の名前空間を表示して、設定が有効かどうか確認します：

   {{< text bash >}}
   $ kubectl config view -o jsonpath="{.contexts[?(@.name==\"$(kubectl config current-context)\")].context.namespace}"
   tutorial
   {{< /text >}}

   出力に講師から割り当てられた、または前のモジュールで自分で割り当てた名前空間が表示されていれば OK です。

1. [Istio リリース](https://github.com/istio/istio/releases) をダウンロードし、
   `bin` ディレクトリからコマンドラインツール `istioctl` を取り出します。下記コマンドで `istioctl` が正しく動作するか確認します：

   {{< text bash >}}
   $ istioctl version
   client version: 1.22.0
   control plane version: 1.22.0
   data plane version: 1.22.0 (4 proxies)
   {{< /text >}}

おめでとうございます。ローカルマシンのセットアップが完了しました！

次は[ローカルでマイクロサービスを実行](/ja/docs/examples/microservices-istio/single/)しましょう。
