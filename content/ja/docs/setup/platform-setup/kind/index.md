---
title: kind
description: Istio 用の kind セットアップ手順。
weight: 30
skip_seealso: true
keywords: [platform-setup, kubernetes, kind]
owner: istio/wg-environments-maintainers
test: no
---

[kind](https://kind.sigs.k8s.io/) は、Docker コンテナの `nodes` を使ってローカル Kubernetes クラスタを実行するためのツールです。
kind は主に Kubernetes 自体のテスト用に設計されていますが、ローカル開発や CI にも利用できます。
以下の手順に従って、Istio インストール用の kind クラスタを準備してください。

## 前提条件 {#prerequisites}

- 最新の Go バージョンを使用してください。
- kind を利用するには、[Docker のインストール](https://docs.docker.com/install/)も必要です。
- 最新バージョンの [kind](https://kind.sigs.k8s.io/docs/user/quick-start/) をインストールしてください。
- Docker の[メモリ制限](/ja/docs/setup/platform-setup/docker/)を増やしてください。

## インストール手順 {#installation-steps}

1.  次のコマンドでクラスタを作成します：

    {{< text bash >}}
    $ kind create cluster --name istio-testing
    {{< /text >}}

    `--name` でクラスタ名を指定できます。デフォルトではクラスタ名は `kind` になります。

1.  次のコマンドで kind クラスタの一覧を表示します：

    {{< text bash >}}
    $ kind get clusters
    istio-testing
    {{< /text >}}

1.  次のコマンドでローカルの Kubernetes 環境を確認します：

    {{< text bash >}}
    $ kubectl config get-contexts
    CURRENT NAME CLUSTER AUTHINFO NAMESPACE

    -         kind-istio-testing   kind-istio-testing   kind-istio-testing
                minikube             minikube             minikube
      {{< /text >}}

    {{< tip >}}
    `kind` が環境名やクラスタ名の先頭に付与されます（例：`kind-istio-testing`）。
    {{< /tip >}}

1.  複数のクラスタを実行している場合、`kubectl` で操作するクラスタを選択する必要があります。
    デフォルトのクラスタは [Kubernetes kubeconfig](https://kubernetes.io/ja/docs/concepts/configuration/organize-cluster-access-kubeconfig/)
    ファイルで現在のコンテキストを設定することで指定できます。また、次のコマンドで `kubectl` の現在のコンテキストを設定できます：

    {{< text bash >}}
    $ kubectl config use-context kind-istio-testing
    Switched to context "kind-istio-testing".
    {{< /text >}}

    kind クラスタのセットアップが完了したら、[Istio リリースをインストール](/ja/docs/setup/additional-setup/download-istio-release/)できます。

1.  検証や実験が終わり、クラスタを削除したい場合は次のコマンドを実行します：

    {{< text bash >}}
    $ kind delete cluster --name istio-testing
    Deleting cluster "istio-testing" ...
    {{< /text >}}

## kind でロードバランサーをセットアップする {#setup-loadbalancer-for-kind}

kind には `Loadbalancer` サービスタイプに IP アドレスを割り当てるための組み込み機能はありません。
`Gateway` サービスに IP アドレスを割り当てるには、[このガイド](https://kind.sigs.k8s.io/docs/user/loadbalancer/)を参照してください。

## kind でダッシュボード UI をセットアップする {#setup-dashboard-ui-for-kind}

kind には minikube のような組み込みのダッシュボード UI はありませんが、
Web ベースの Kubernetes ダッシュボードをセットアップしてクラスタを確認できます。
以下の手順で kind 用のダッシュボードをセットアップします。

1. 次のコマンドでダッシュボードをデプロイします：

   {{< text bash >}}
   $ kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.7.0/aio/deploy/recommended.yaml
   {{< /text >}}

1. ダッシュボードがデプロイされ、稼働していることを確認します。

   {{< text bash >}}
   $ kubectl get pod -n kubernetes-dashboard
   NAME READY STATUS RESTARTS AGE
   dashboard-metrics-scraper-76585494d8-zdb66 1/1 Running 0 39s
   kubernetes-dashboard-b7ffbc8cb-zl8zg 1/1 Running 0 39s
   {{< /text >}}

1. 新しく作成したクラスタに管理者権限を与えるため、`ServiceAccount` と `ClusterRoleBinding` を作成します。

   {{< text bash >}}
   $ kubectl create serviceaccount -n kubernetes-dashboard admin-user
   $ kubectl create clusterrolebinding -n kubernetes-dashboard admin-user --clusterrole cluster-admin --serviceaccount=kubernetes-dashboard:admin-user
   {{< /text >}}

1. ダッシュボードにログインするには、ベアラートークンが必要です。次のコマンドでトークンを変数に格納します。

   {{< text bash >}}
   $ token=$(kubectl -n kubernetes-dashboard create token admin-user)
   {{< /text >}}

   `echo` コマンドでトークンを表示し、ダッシュボードへのログインに使用するためコピーしてください。

   {{< text bash >}}
   $ echo $token
   {{< /text >}}

1. kubectl コマンドラインツールを使ってダッシュボードにアクセスするには、次のコマンドを実行します：

   {{< text bash >}}
   $ kubectl proxy
   Starting to serve on 127.0.0.1:8001
   {{< /text >}}

   [Kubernetes ダッシュボード](http://localhost:8001/api/v1/namespaces/kubernetes-dashboard/services/https:kubernetes-dashboard:/proxy/) をクリックして、デプロイやサービスを確認できます。

   {{< warning >}}
   トークンはどこかに保存しておいてください。そうしないと、ダッシュボードにログインするたびに 4 番目の手順を実行する必要があります。
   {{< /warning >}}
