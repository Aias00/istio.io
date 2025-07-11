---
title: k3d
description: Istio 用の k3d セットアップ手順。
weight: 28
skip_seealso: true
keywords: [platform-setup, kubernetes, k3d, k3s]
owner: istio/wg-environments-maintainers
test: no
---

k3d は、[k3s](https://github.com/rancher/k3s)（Rancher Lab の最小限の Kubernetes ディストリビューション）を Docker 上で実行するための軽量ラッパーです。
k3d を使うと、Docker 上で単一ノードや複数ノードの k3s クラスタを簡単に作成でき、Kubernetes のローカル開発などに便利です。

## 前提条件 {#prerequisites}

- k3d を利用するには、[Docker のインストール](https://docs.docker.com/install/)も必要です。
- 最新バージョンの [k3d](https://k3d.io/v5.4.7/#installation) をインストールしてください。
- Kubernetes クラスタとやり取りするための [kubectl](https://kubernetes.io/ja/docs/tasks/tools/#kubectl)。
- （オプション）[Helm](https://helm.sh/docs/intro/install/) は Kubernetes 用のパッケージマネージャです。

## インストール {#installation}

1.  次のコマンドでクラスタを作成し、`Traefik` を無効化します：

    {{< text bash >}}
    $ k3d cluster create --api-port 6550 -p '9080:80@loadbalancer' -p '9443:443@loadbalancer' --agents 2 --k3s-arg '--disable=traefik@server:\*'
    {{< /text >}}

1.  k3d クラスタの一覧を表示するには、次のコマンドを実行します：

    {{< text bash >}}
    $ k3d cluster list
    k3s-default
    {{< /text >}}

1.  ローカルの Kubernetes コンテキストを一覧表示するには、次のコマンドを実行します。

    {{< text bash >}}
    $ kubectl config get-contexts
    CURRENT NAME CLUSTER AUTHINFO NAMESPACE

    -         k3d-k3s-default      k3d-k3s-default      k3d-k3s-default
      {{< /text >}}

    {{< tip >}}
    `k3d-` がコンテキスト名やクラスタ名の先頭に付与されます（例：`k3d-k3s-default`）。
    {{< /tip >}}

1.  複数のクラスタを実行している場合、`kubectl` がどのクラスタと対話するかを選択する必要があります。デフォルトのクラスタは、
    [Kubernetes kubeconfig](https://kubernetes.io/ja/docs/concepts/configuration/organize-cluster-access-kubeconfig/)
    ファイルで現在のコンテキストを設定することで指定できます。また、次のコマンドで `kubectl` の現在のコンテキストを設定できます。

    {{< text bash >}}
    $ kubectl config use-context k3d-k3s-default
    Switched to context "k3d-k3s-default".
    {{< /text >}}

## k3d での Istio セットアップ {#set-up-istio-for-k3d}

1. k3d クラスタのセットアップが完了したら、[Helm 3 を使って Istio をインストール](/ja/docs/setup/install/helm/)できます。

   {{< text bash >}}
   $ kubectl create namespace istio-system
   $ helm install istio-base istio/base -n istio-system --wait
   $ helm install istiod istio/istiod -n istio-system --wait
   {{< /text >}}

1. （オプション）Ingress Gateway をインストールします：

   {{< text bash >}}
   $ helm install istio-ingressgateway istio/gateway -n istio-system --wait
   {{< /text >}}

## k3d でダッシュボード UI をセットアップする {#set-up-dashboard-UI-for-k3d}

k3d には minikube のような組み込みのダッシュボード UI はありませんが、
Web ベースの Kubernetes UI である Dashboard をセットアップしてクラスタを確認できます。
以下の手順で k3d 用のダッシュボードをセットアップします。

1. ダッシュボードをデプロイするには、次のコマンドを実行します：

   {{< text bash >}}
   $ GITHUB_URL=https://github.com/kubernetes/dashboard/releases
   $ VERSION_KUBE_DASHBOARD=$(curl -w '%{url_effective}' -I -L -s -S ${GITHUB_URL}/latest -o /dev/null | sed -e 's|.*/||')
    $ kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/${VERSION_KUBE_DASHBOARD}/aio/deploy/recommended.yaml
   {{< /text >}}

1. ダッシュボードがデプロイされ、稼働していることを確認します。

   {{< text bash >}}
   $ kubectl get pod -n kubernetes-dashboard
   NAME READY STATUS RESTARTS AGE
   dashboard-metrics-scraper-8c47d4b5d-dd2ks 1/1 Running 0 25s
   kubernetes-dashboard-67bd8fc546-4xfmm 1/1 Running 0 25s
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

## アンインストール {#uninstall}

1. 実験が終わり既存のクラスタを削除したい場合は、次のコマンドを実行します：

   {{< text bash >}}
   $ k3d cluster delete k3s-default
   Deleting cluster "k3s-default" ...
   {{< /text >}}
