---
title: IBM Cloud クイックスタート
description: IBM パブリッククラウドまたはプライベートクラウドで Istio サービスを素早くセットアップ。
weight: 25
skip_seealso: true
aliases:
  - /zh/docs/setup/kubernetes/prepare/platform-setup/ibm/
  - /zh/docs/setup/kubernetes/platform-setup/ibm/
keywords: [platform-setup, ibm, iks]
owner: istio/wg-environments-maintainers
test: no
---

以下の手順に従って、[IBM Cloud Kubernetes Service](https://cloud.ibm.com/docs/containers?topic=containers-getting-started) パブリッククラウド上で Istio サービスクラスタをセットアップします。
[Istio on IBM Cloud Private](https://www.ibm.com/support/knowledgecenter/en/SSBS6K_3.2.1/manage_cluster/istio.html) プライベートクラウド上で Istio サービスクラスタをセットアップします。

{{< tip >}}
IBM は IBM Cloud Kubernetes Service 用の {{< gloss >}}マネージドコントロールプレーン{{< /gloss >}} プラグインを提供しています。
このプラグインを使うことで、Istio を手動でインストールする代わりに利用できます。詳細や手順については、
[Istio on IBM Cloud Kubernetes Service](https://cloud.ibm.com/docs/containers?topic=containers-istio) をご参照ください。
{{< /tip >}}

手動で Istio をインストールする前にクラスタを準備するには、以下の手順に従ってください：

1. [IBM Cloud CLI、IBM Cloud Kubernetes Service プラグイン、Kubernetes CLI のインストール](https://cloud.ibm.com/docs/containers?topic=containers-cs_cli_install)。

1. 以下のコマンドで標準 Kubernetes クラスタを作成します。
   `<cluster-name>` をクラスタ名に、`<zone-name>` を利用可能なゾーン名に置き換えてください。

   {{< tip >}}
   利用可能なゾーンは `ibmcloud ks zones` で表示できます。
   IBM Cloud Kubernetes Service [ロケーションリファレンスガイド](https://cloud.ibm.com/docs/containers?topic=containers-regions-and-zones)
   では利用可能なゾーンや指定方法について説明しています。
   {{< /tip >}}

   {{< text bash >}}
   $ ibmcloud ks cluster create classic --zone <zone-name> --machine-type b3c.4x16 \
    --workers 3 --name <cluster-name>
   {{< /text >}}

   {{< tip >}}
   すでに専用 VLAN とパブリック VLAN がある場合は、上記コマンドで `--private-vlan` と `--public-vlan` オプションを指定できます。
   ない場合は自動的に作成されます。`ibmcloud ks vlans --zone <zone-name>` で利用可能な VLAN を確認できます。
   {{< /tip >}}

1. 次のコマンドで `kubectl` クラスタ設定をダウンロードし、出力された指示に従って `KUBECONFIG` 環境変数を設定してください。

   {{< text bash >}}
   $ ibmcloud ks cluster config --cluster <cluster-name>
   {{< /text >}}

   {{< warning >}}
   クラスタの Kubernetes バージョンに合った `kubectl` CLI バージョンを必ず使用してください。
   {{< /warning >}}
