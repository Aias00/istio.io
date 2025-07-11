---
title: アリババクラウド
description: アリババクラウド Kubernetes クラスタで Istio をインストール・運用するための設定。
weight: 5
skip_seealso: true
aliases:
  - /zh/docs/setup/kubernetes/prepare/platform-setup/alicloud/
  - /zh/docs/setup/kubernetes/platform-setup/alicloud/
keywords: [platform-setup, alibaba-cloud, aliyun, alicloud]
owner: istio/wg-environments-maintainers
test: n/a
---

このページの最終更新日は 2018 年 8 月 8 日です。

{{< boilerplate untested-document >}}

以下の手順に従って、[アリババクラウド Kubernetes コンテナサービス](https://www.alibabacloud.com/zh/product/kubernetes)クラスタを設定し、Istio をインストール・運用できるようにします。
アリババクラウドの **コンテナサービス管理コンソール** から、Istio を完全にサポートする Kubernetes クラスタを簡単にデプロイできます。

{{< tip >}}
アリババクラウドは、Istio と完全互換のマネージドサービスメッシュプラットフォーム「アリババクラウドサービスメッシュ（ASM）」を提供しています。詳細や手順については、
[アリババクラウドサービスメッシュ](https://www.alibabacloud.com/help/zh/alibaba-cloud-service-mesh/latest/what-is-asm) をご参照ください。
{{< /tip >}}

## 前提条件{#prerequisites}

1. [アリババクラウドの手順](https://www.alibabacloud.com/help/zh/container-service-for-kubernetes/latest/create-an-ack-managed-cluster)に従い、以下のサービスを有効化してください：
   コンテナサービス、リソースオーケストレーションサービス（ROS）、および RAM。

## 手順{#procedure}

1. `コンテナサービス管理コンソール` にログインし、左側のナビゲーションバーの **Kubernetes** 配下にある **クラスタ** をクリックして **クラスタ一覧** ページに移動します。

1. 右上の **Kubernetes クラスタ作成** ボタンをクリックします。

1. クラスタ名を入力します。クラスタ名は 1 ～ 63 文字で、数字・中国語・英字・ハイフン（-）を含めることができます。

1. クラスタの **region** と **zone** を選択します。

1. クラスタネットワークタイプを設定します。Kubernetes クラスタは現在 VPC ネットワークタイプのみサポートしています。

1. ノードタイプを設定します。従量課金とサブスクリプション（年単位・月単位）に対応しています。

1. マスターノードを設定します。マスターノードのインスタンスタイプを選択します。

1. ワーカーノードを設定します。ワーカーノードを新規作成するか、既存の ECS インスタンスをワーカーノードとして追加するかを選択します。

1. ログイン方式、Pod のネットワーク CIDR、Service CIDR を設定します。

下図は、上記のすべての手順を完了した画面例です：

{{< image link="./csconsole.png" caption="コンソール" >}}
