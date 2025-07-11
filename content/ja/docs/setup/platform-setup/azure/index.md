---
title: Azure
description: Istio 用の Azure クラスタをセットアップするための手順。
weight: 10
skip_seealso: true
aliases:
  - /zh/docs/setup/kubernetes/prepare/platform-setup/azure/
  - /zh/docs/setup/kubernetes/platform-setup/azure/
keywords: [platform-setup, azure]
owner: istio/wg-environments-maintainers
test: no
---

以下の手順に従って、Istio 用の Azure クラスタを準備してください。

{{< tip >}}
Azure では Azure Kubernetes Service (AKS) 用の
{{< gloss >}}マネージドコントロールプレーン{{< /gloss >}}アドオンが提供されています。
これを使うことで、Istio を手動でインストールする代わりに利用できます。詳細やチュートリアルは
[Azure Kubernetes Service での Istio ベースのサービスメッシュアドオンのデプロイ](https://learn.microsoft.com/zh-cn/azure/aks/istio-deploy-addon) をご参照ください。
{{< /tip >}}

[AKS](https://azure.microsoft.com/zh-cn/services/kubernetes-service/)（Istio を完全サポート）や、
[自前の Kubernetes または AKS で使われる Azure クラスタ API プロバイダー（CAPZ）](https://capz.sigs.k8s.io/) を使って、Azure 上に Kubernetes クラスタをデプロイできます。

## AKS

AKS クラスタは、
[az cli](https://docs.microsoft.com/zh-cn/azure/aks/kubernetes-walkthrough)、
[Azure ポータル](https://docs.microsoft.com/zh-cn/azure/aks/kubernetes-walkthrough-portal)、
[az cli with Bicep](https://learn.microsoft.com/zh-cn/azure/aks/learn/quick-kubernetes-deploy-bicep?tabs=azure-cli)
または [Terraform](https://learn.microsoft.com/zh-cn/azure/aks/learn/quick-kubernetes-deploy-terraform?tabs=bash) など、さまざまな方法で作成できます。

`az` cli を使う場合は、`az login` で認証するか、Cloud Shell で以下のコマンドを実行してください。

1. AKS をサポートする対象リージョン名を確認します。

   {{< text bash >}}
   $ az provider list --query "[?namespace=='Microsoft.ContainerService'].resourceTypes[] | [?resourceType=='managedClusters'].locations[]" -o tsv
   {{< /text >}}

1. 対象リージョンでサポートされている Kubernetes バージョンを確認します。

   前のステップで取得したリージョン値で `my location` を置き換えて実行します：

   {{< text bash >}}
   $ az aks get-versions --location "my location" --query "orchestrators[].orchestratorVersion"
   {{< /text >}}

1. リソースグループを作成し、AKS クラスタをデプロイします。

   ステップ 1 で取得した `mylocation` 名で `myResourceGroup` と `myAKSCluster` を置き換えてください。
   そのリージョンで `Kubernetes 1.28.3` がサポートされていない場合は、次のコマンドを実行します：

   {{< text bash >}}
   $ az group create --name myResourceGroup --location "my location"
   $ az aks create --resource-group myResourceGroup --name myAKSCluster --node-count 3 --kubernetes-version 1.28.3 --generate-ssh-keys
   {{< /text >}}

1. AKS の `kubeconfig` 資格情報を取得します。

   先ほどの手順で取得した名前で `myResourceGroup` と `myAKSCluster` を置き換えて実行します：

   {{< text bash >}}
   $ az aks get-credentials --resource-group myResourceGroup --name myAKSCluster
   {{< /text >}}
