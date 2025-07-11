---
title: Oracle Cloud 基础架构
description: Oracle Container を使った Istio クラスタ準備ガイド。
weight: 60
skip_seealso: true
aliases:
  - /zh/docs/setup/kubernetes/prepare/platform-setup/oci/
  - /zh/docs/setup/kubernetes/platform-setup/oci/
keywords: [platform-setup, kubernetes, oke, oci, oracle]
owner: istio/wg-environments-maintainers
test: no
---

このページの最終更新日は 2021 年 9 月 20 日です。

{{< boilerplate untested-document >}}

以下の説明に従って、Istio 用の OKE クラスタ環境を構成します。

## OKE クラスタの作成 {#create-an-oke-cluster}

OKE クラスタを作成するには、テナントの管理者であるか、`CLUSTER_MANAGE` 権限を持つグループに属している必要があります。

[OKE クラスタの作成][Create]で最も簡単な方法は、[クイック作成ワークフロー][Quick]を [Oracle Cloud Infrastructure (OCI) コンソール][Console] で利用することです。
その他の方法としては、[カスタム作成ワークフロー][Custom] や [Oracle Cloud Infrastructure (OCI) API][API] があります。

[OCI CLI][OCICLI] を使って、以下の例のようにコマンドラインからクラスタを作成することもできます：

{{< text bash >}}
$ oci ce cluster create \
 --name <oke-cluster-name> \
 --kubernetes-version <kubernetes-version> \
 --compartment-id <compartment-ocid> \
 --vcn-id <vcn-ocid>
{{< /text >}}

| パラメータ           | 説明                                                                 |
| -------------------- | -------------------------------------------------------------------- |
| `oke-cluster-name`   | 新しい OKE クラスタに割り当てる名前                                  |
| `kubernetes-version` | デプロイする[サポートされている Kubernetes バージョン][K8S]          |
| `compartment-ocid`   | 既存の[コンパートメント][CONCEPTS]の [OCID][CONCEPTS]                |
| `vcn-ocid`           | 既存の[仮想クラウドネットワーク][CONCEPTS] (VCN) の [OCID][CONCEPTS] |

## OKE クラスタへのローカルアクセスの設定 {#setting-up-local-access-to-an-OKE-cluster}

ローカルマシンからクラスタにアクセスするには、[kubectl][kubectl] と [OCICLI][OCICLI] (`OCI`) をインストールしてください。

以下の OCI CLI コマンドを使って `kubeconfig` ファイルを作成または更新します。このコマンドは、
`oci` コマンドを含み、短期間有効な認証トークンを動的に生成・挿入して `kubectl` からクラスタへアクセスできるようにします：

{{< text bash >}}
$ oci ce cluster create-kubeconfig \
 --cluster-id <cluster-ocid> \
 --file $HOME/.kube/config \
 --token-version 2.0.0 \
 --kube-endpoint [PRIVATE_ENDPOINT|PUBLIC_ENDPOINT]
{{< /text >}}

{{< tip >}}
OKE クラスタは複数のエンドポイントを公開できますが、`kubeconfig` ファイルには 1 つのエンドポイントのみが記載されます。
{{< /tip >}}

`kube-endpoint` で指定できる値は `PUBLIC_ENDPOINT` または `PRIVATE_ENDPOINT` です。
プライベートエンドポイントのみのクラスタにアクセスする場合は、[Bastion ホスト][bastion] を使って SSH トンネルを構成する必要がある場合があります。

`cluster-ocid` には対象の OKE クラスタの [OCID][CONCEPTS] を指定してください。

## クラスタへのアクセス確認 {#verify-access-to-the-cluster}

`kubectl get nodes` コマンドで `kubectl` がクラスタに接続できるか確認します：

{{< text bash >}}
$ kubectl get nodes
{{< /text >}}

これで [`istioctl`](../../install/istioctl/)、(Helm)(../../install/helm/) または手動で Istio をインストールできます。

[CREATE]: https://docs.oracle.com/en-us/iaas/Content/ContEng/Tasks/contengcreatingclusterusingoke.htm
[API]: https://docs.oracle.com/en-us/iaas/Content/ContEng/Tasks/contengcreatingclusterusingoke_topic-Using_the_API.htm
[QUICK]: https://docs.oracle.com/en-us/iaas/Content/ContEng/Tasks/contengcreatingclusterusingoke_topic-Using_the_Console_to_create_a_Quick_Cluster_with_Default_Settings.htm
[CUSTOM]: https://docs.oracle.com/en-us/iaas/Content/ContEng/Tasks/contengcreatingclusterusingoke_topic-Using_the_Console_to_create_a_Custom_Cluster_with_Explicitly_Defined_Settings.htm
[OCICLI]: https://docs.oracle.com/en-us/iaas/Content/API/SDKDocs/cliinstall.htm
[K8S]: https://docs.oracle.com/en-us/iaas/Content/ContEng/Concepts/contengaboutk8sversions.htm
[KUBECTL]: https://kubernetes.io/zh-cn/docs/tasks/tools/
[CONCEPTS]: https://docs.oracle.com/en-us/iaas/Content/GSG/Concepts/concepts.htm
[BASTION]: https://docs.oracle.com/en-us/iaas/Content/ContEng/Tasks/contengdownloadkubeconfigfile.htm#localdownload
[CONSOLE]: https://docs.oracle.com/en-us/iaas/Content/GSG/Concepts/console.htm
