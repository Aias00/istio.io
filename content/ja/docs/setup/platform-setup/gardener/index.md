---
title: Kubernetes Gardener クイックスタート
description: Gardener を使って Istio サービスを素早くセットアップ。
weight: 35
aliases:
  - /zh/docs/setup/kubernetes/platform-setup/gardener/
skip_seealso: true
keywords: [platform-setup, kubernetes, gardener, sap]
owner: istio/wg-environments-maintainers
test: no
---

## Gardener ブートストラップ {#bootstrapping-gardener}

自分の組織の Kubernetes サービス要件を満たすために [Gardener](https://gardener.cloud) をセットアップしたい場合は、[ドキュメント](https://github.com/gardener/gardener/blob/master/docs/README.md) を参照してください。
テスト目的の場合は、リポジトリをクローンして `make kind-up gardener-up` を実行することで
[ローカルで Gardener をセットアップ](https://github.com/gardener/gardener/blob/master/docs/development/getting_started_locally.md) できます（これは開発時に Gardener を呼び出す最も簡単な方法です）。

また、[`23 Technologies GmbH`](https://23technologies.cloud/) は完全マネージドの Gardener サービスを提供しており、
すべてのサポートされているクラウドプロバイダーに簡単に対応でき、無料トライアルもあります：[`Okeanos`](https://okeanos.dev/)。
同様に、[`STACKIT`](https://stackit.de/)、[`B'Nerd`](https://bnerd.com/)、[`MetalStack`](https://metalstack.cloud/)
など多くのクラウドプロバイダーが Gardener を Kubernetes エンジンとして利用できます。

オープンソースプロジェクトについて詳しく知りたい場合は、[`kubernetes.io`](https://kubernetes.io/zh-cn/blog) の
[Gardener プロジェクトアップデート](https://kubernetes.io/blog/2019/12/02/gardener-project-update/) や
[Gardener - Kubernetes 植物学者](https://kubernetes.io/blog/2018/05/17/gardener/) をご覧ください。

[Istio、カスタムドメイン、証明書で自分だけの Gardener を素早く使う](https://gardener.cloud/docs/extensions/others/gardener-extension-shoot-cert-service/tutorials/tutorial-custom-domain-with-istio/)
は、Gardener のエンドユーザー向けの詳細なチュートリアルです。

### `kubectl` のインストールと設定 {#install-and-configure-Kubernetes}

1. すでに `kubectl` CLI をお持ちの場合は、`kubectl version --short` を実行してバージョンを確認してください。
   利用する Kubernetes クラスタのバージョンと同等以上のバージョンが必要です。
   `kubectl` のバージョンが古い場合は、次の手順で新しいバージョンをインストールしてください。

1. [`kubectl` CLI をインストール](https://kubernetes.io/zh-cn/docs/tasks/tools/) してください。

### Gardener へのアクセス {#access-gardener}

1. Gardener ダッシュボードでプロジェクトを作成します。これにより、`garden-<my-project>` という名前の Kubernetes ネームスペースが作成されます。

1. [Gardener プロジェクトへのアクセス権を kubeconfig で設定](https://gardener.cloud/docs/dashboard/usage/gardener-api/) します。

   {{< tip >}}
   Gardener ダッシュボードと組み込みの Web ターミナルでクラスタを作成・操作する場合はこの手順をスキップできます。プログラムによるアクセスのみ必要です。
   {{< /tip >}}

   まだ Gardener 管理者でない場合は、Gardener ダッシュボードでテクニカルユーザーを作成できます。
   "Members" セクションに移動し、サービスアカウントを追加してください。その後、プロジェクト用の kubeconfig をダウンロードできます。
   シェルで `export KUBECONFIG=garden-my-project.yaml` を設定してください。

   ![Gardener で kubeconfig をダウンロード](https://raw.githubusercontent.com/gardener/dashboard/master/docs/images/01-add-service-account.png "サービスアカウントで kubeconfig をダウンロード")

### Kubernetes クラスタの作成 {#creating-a-Kubernetes-cluster}

クラスタ仕様の yaml ファイルを用意し、`kubectl` CLI でクラスタを作成できます。
[GCP 用のサンプルはこちら](https://github.com/gardener/gardener/blob/master/example/90-shoot.yaml)。
ネームスペースがプロジェクトのネームスペースと一致していることを確認してください。準備した "shoot" クラスタマニフェストを `kubectl` で適用するだけです：

{{< text bash >}}
$ kubectl apply --filename my-cluster.yaml
{{< /text >}}

より簡単な方法として、Gardener ダッシュボードのクラスタ作成ウィザードに従ってクラスタを作成することもできます：

![shoot クラスタの作成](https://raw.githubusercontent.com/gardener/dashboard/master/docs/images/dashboard-demo.gif "ダッシュボードで shoot クラスタを作成")

### クラスタ用 `kubectl` の設定 {#configure-Kubernetes-for-your-cluster}

今、Gardener ダッシュボードまたは CLI で新しく作成したクラスタ用の kubeconfig をダウンロードできます。例：

{{< text bash >}}
$ kubectl --namespace shoot--my-project--my-cluster get secret kubecfg --output jsonpath={.data.kubeconfig} | base64 --decode > my-cluster.yaml
{{< /text >}}

この kubeconfig ファイルで管理者はクラスタにフルアクセスできます。
ワークロードクラスタで作業する際は、`export KUBECONFIG=my-cluster.yaml` を設定してください。

## 削除 {#cleaning-up}

Gardener ダッシュボードでクラスタを削除するか、`garden-my-project.yaml` kubeconfig を指定して `kubectl` で以下を実行します：

{{< text bash >}}
$ kubectl --kubeconfig garden-my-project.yaml --namespace garden--my-project annotate shoot my-cluster confirmation.garden.sapcloud.io/deletion=true
$ kubectl --kubeconfig garden-my-project.yaml --namespace garden--my-project delete shoot my-cluster
{{< /text >}}
