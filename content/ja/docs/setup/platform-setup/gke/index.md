---
title: Google Kubernetes Engine クイックスタート
description: Google Kubernetes Engine (GKE) で Istio サービスを素早くセットアップ。
weight: 20
skip_seealso: true
aliases:
  - /zh/docs/setup/kubernetes/prepare/platform-setup/gke/
  - /zh/docs/setup/kubernetes/platform-setup/gke/
keywords: [platform-setup, kubernetes, gke, google]
owner: istio/wg-environments-maintainers
test: no
---

以下の手順に従って、Istio 用の GKE クラスタを準備してください。

1. 新しいクラスタを作成します。

   {{< text bash >}}
   $ export PROJECT_ID=`gcloud config get-value project` && \
    export M_TYPE=n1-standard-2 && \
    export ZONE=us-west2-a && \
    export CLUSTER_NAME=${PROJECT_ID}-${RANDOM} && \
    gcloud services enable container.googleapis.com && \
    gcloud container clusters create $CLUSTER_NAME \
      --cluster-version latest \
      --machine-type=$M_TYPE \
    --num-nodes 4 \
    --zone $ZONE \
    --project $PROJECT_ID
   {{< /text >}}

   {{< tip >}}
   Istio のデフォルトインストールではノードの vCPU が 1 より大きい必要があります。[構成プロファイルの例](/zh/docs/setup/additional-setup/config-profiles/) を使う場合は、
   `--machine-type` パラメータを省略して、より小さい `n1-standard-1` マシンタイプを使うこともできます。
   {{< /tip >}}

   {{< warning >}}
   GKE Standard で Istio CNI 機能を使う場合は、
   [CNI インストールガイド](/zh/docs/setup/additional-setup/cni/#prerequisites) の前提条件クラスタ設定手順を参照してください。
   CNI ノードエージェントは SYS_ADMIN 機能が必要なため、GKE Autopilot では利用できません。istio-init コンテナを使用してください。
   {{< /warning >}}

   {{< warning >}}
   **プライベート GKE クラスタ**

   istiod の Validation Webhook には 15017 ポートが必要ですが、自動作成されるファイアウォールルールではこのポートが開放されません。

   以下の手順でファイアウォールルールを確認し、Master からのアクセスを許可してください：

   {{< text bash >}}
   $ gcloud compute firewall-rules list --filter="name~gke-${CLUSTER_NAME}-[0-9a-z]\*-master"
   {{< /text >}}

   現在のファイアウォールルールを置き換えて Master からのアクセスを許可します：

   {{< text bash >}}
   $ gcloud compute firewall-rules update <firewall-rule-name> --allow tcp:10250,tcp:443,tcp:15017
   {{< /text >}}

   {{< /warning >}}

1. `kubectl` の認証情報を取得します。

   {{< text bash >}}
   $ gcloud container clusters get-credentials <cluster-name> \
    --zone <zone> \
    --project <project-id>
   {{< /text >}}

1. Istio 用の RBAC ルールを作成します。現在のユーザーにクラスタ管理者（admin）権限を付与するには、以下のコマンドを実行してください。

   {{< text bash >}}
   $ kubectl create clusterrolebinding cluster-admin-binding \
    --clusterrole=cluster-admin \
    --user=$(gcloud config get-value core/account)
   {{< /text >}}

## マルチクラスタ通信 {#multi-cluster-communication}

場合によっては、クラスタ間通信を許可するために明示的にファイアウォールルールを作成する必要があります。

{{< warning >}}
以下の手順は、プロジェクト内の**すべて**のクラスタ間通信を許可します。必要に応じてコマンドを調整してください。
{{< /warning >}}

1. クラスタネットワーク情報を収集します。

   {{< text bash >}}
   $ function join_by { local IFS="$1"; shift; echo "$\*"; }
   $ ALL_CLUSTER_CIDRS=$(gcloud --project $PROJECT_ID container clusters list --format='value(clusterIpv4Cidr)' | sort | uniq)
    $ ALL_CLUSTER_CIDRS=$(join_by , $(echo "${ALL_CLUSTER_CIDRS}"))
   $ ALL_CLUSTER_NETTAGS=$(gcloud --project $PROJECT_ID compute instances list --format='value(tags.items.[0])' | sort | uniq)
    $ ALL_CLUSTER_NETTAGS=$(join_by , $(echo "${ALL_CLUSTER_NETTAGS}"))
   {{< /text >}}

1. ファイアウォールルールを作成します。

   {{< text bash >}}
   $ gcloud compute firewall-rules create istio-multicluster-pods \
    --allow=tcp,udp,icmp,esp,ah,sctp \
    --direction=INGRESS \
    --priority=900 \
    --source-ranges="${ALL_CLUSTER_CIDRS}" \
        --target-tags="${ALL_CLUSTER_NETTAGS}" --quiet
   {{< /text >}}
