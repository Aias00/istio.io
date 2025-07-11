---
title: 腾讯云
description: Tencent Cloud 上での Istio サービスのクイック作成。
weight: 65
skip_seealso: true
keywords: [platform-setup, tencent-cloud-mesh, tcm, tencent-cloud, tencentcloud]
owner: istio/wg-environments-maintainers
test: n/a
---

## 前提作業 {#prerequisites}

以下の手順に従って、Istio 用の [Tencent Kubernetes Engine](https://cloud.tencent.com/product/tke)
または [Elastic Kubernetes Service](https://cloud.tencent.com/product/eks) クラスタをセットアップしてください。

Tencent Cloud では [Tencent Kubernetes Engine](https://cloud.tencent.com/document/product/457/32189)
または [Elastic Kubernetes Service](https://cloud.tencent.com/document/product/457/39813)
を使って Kubernetes クラスタをデプロイできます。どちらのクラスタも Istio のインストールとデプロイに完全対応しています。

{{< image link="./tke.png" caption="クラスタの作成" >}}

## 手順 {#procedure}

TKE または EKS クラスタを作成した後、[Tencent Cloud Mesh](https://cloud.tencent.com/product/tcm)
で Istio を素早くデプロイ・利用できます。

{{< image link="./tcm.png" caption="Tencent サービスメッシュの作成" >}}

1. **コンテナサービス**のコンソールにログインし、左側のナビゲーションバーから**サービスメッシュ**をクリックして**サービスメッシュ**ページに移動します。

1. 左上の**作成**ボタンをクリックします。

1. Mesh の名前を入力します。

   {{< tip >}}
   Mesh 名は 1 ～ 60 文字で、数字・日本語・英字・ハイフン（-）を含めることができます。
   {{< /tip >}}

1. クラスタの**リージョン**を選択します。

1. インストールする Istio のバージョンを選択します。Tencent Cloud Mesh では最新の 2 つの主要バージョンの Istio をサポートしています。

1. サービスメッシュのデプロイモードを選択します：**独立メッシュ**または**マネージドメッシュ**。

   {{< tip >}}
   Tencent Cloud Mesh は**独立メッシュモード**（Istiod がユーザークラスタ内で動作し、ユーザー自身が管理）と、
   **マネージドメッシュモード**（Istiod が管理プレーンで動作し、Tencent Cloud Mesh チームが管理）をサポートしています。
   {{< /tip >}}

1. Egress のトラフィックルールを設定します：`Register Only` または `Allow Any`。

1. 関連する **Tencent Kubernetes Engine** または **Elastic Kubernetes Service** クラスタを選択します。

1. 指定した Namespaces で Sidecar の自動インジェクションを有効にします。

1. 外部リクエストが Sidecar をバイパスして直接アクセスできる IP アドレスブロックを設定します。外部リクエストのトラフィックは Istio
   のトラフィック管理や可観測性などの機能を利用できなくなります。デフォルトではすべての外部リクエストは Sidecar 経由で転送されます。

1. Sidecar レディネス保証を有効にします。

   {{< tip >}}
   有効にすると、ビジネスコンテナは Sidecar の準備ができてから起動します。これにより Pod の起動時間が若干長くなりますが、
   Sidecar 機能に強く依存するサービスには有効化を推奨します。
   {{< /tip >}}

   {{< image link="./ingress-egress.png" caption="Gateway の設定" >}}

1. エッジプロキシゲートウェイを設定し、Ingress Gateway または Egress Gateway を有効にします。

   {{< image link="./tps.png" caption="可観測性サービスの設定" >}}

1. Metrics、Tracing、Logging などの可観測性機能を設定します。

   {{< tip >}}
   デフォルトのクラウド監視サービスに加え、
   [Prometheus 監視サービス](https://cloud.tencent.com/product/tmp)
   や[ログサービス](https://cloud.tencent.com/product/cls)などの高度な外部サービスも有効化できます。
   {{< /tip >}}

これらの手順を完了し、設定を確認して Istio を作成すれば、Tencent Cloud Mesh で Istio を利用開始できます。
