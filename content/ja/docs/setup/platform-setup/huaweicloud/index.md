---
title: Huawei Cloud
description: Istio 用の Huawei Cloud Kubernetes クラスタをセットアップする手順。
weight: 23
skip_seealso: true
aliases:
  - /zh/docs/setup/kubernetes/prepare/platform-setup/huaweicloud/
  - /zh/docs/setup/kubernetes/platform-setup/huaweicloud/
keywords: [platform-setup, huawei, huaweicloud, cce]
owner: istio/wg-environments-maintainers
test: no
---

以下の手順に従って、[Huawei Cloud コンテナエンジン CCE](https://www.huaweicloud.com/intl/zh-cn/product/cce.html) クラスタを設定し、Istio をインストール・運用できるようにします。
Huawei Cloud の`コンテナエンジンコンソール`から、Istio を完全にサポートする Kubernetes クラスタを簡単にデプロイできます。

{{< tip >}}
Huawei は Huawei Cloud コンテナエンジン CCE 用の{{< gloss >}}マネージドコントロールプレーン{{< /gloss >}}プラグインを提供しています。
このプラグインを使うことで、Istio を手動でインストールする代わりに利用できます。詳細や手順については、
[Huawei アプリケーションサービスメッシュ](https://support.huaweicloud.com/asm/index.html) をご参照ください。
{{< /tip >}}

[Huawei Cloud の手順](https://support.huaweicloud.com/qs-cce/cce_qs_0008.html)に従ってクラスタを準備し、以下の手順で Istio を手動インストールしてください：

1.  CCE コンソールにログインします。**Dashboard** > **クラスタ購入** を選択して **ハイブリッドクラスタ購入** ページを開きます。
    または、ナビゲーションペインで **リソース管理** > **クラスタ** を選択し、**ハイブリッドクラスタ** の横にある **購入** をクリックします。

1.  **クラスタ構成** ページでクラスタパラメータを設定します。以下の例では、ほとんどのパラメータはデフォルトのままです。クラスタ構成が完了したら、
    **次へ** をクリックし、**ノード作成** ページに進みます。

    {{< tip >}}
    Istio には Kubernetes バージョンの要件があります。Istio の[サポートポリシー](/zh/docs/releases/supported-releases#support-status-of-istio-releases)に従ってバージョンを選択してください。
    {{< /tip >}}

    下図はクラスタの作成・構成 GUI 例です：

    {{< image link="./create-cluster.png" caption="クラスタ構成" >}}

1.  ノード作成ページで、以下のパラメータを設定します。

    {{< tip >}}
    Istio は追加のリソース消費があるため、最低でも 4 vCPU と 8 GB メモリを確保することを推奨します。
    {{< /tip >}}

    下図はノードの作成・構成 GUI 例です：

    {{< image link="./create-node.png" caption="ノード構成" >}}

1.  [kubectl の設定](https://support.huaweicloud.com/intl/zh-cn/cce_faq/cce_faq_00041.html)

1.  これで[インストールガイド](/zh/docs/setup/install)に従って CCE クラスタに Istio をインストールできます。

1.  [ELB](https://support.huaweicloud.com/intl/productdesc-elb/en-us_topic_0015479966.html) を設定して、Istio イングレスゲートウェイを公開します（必要な場合）。

    - [弾性ロードバランサーの作成](https://console.huaweicloud.com/vpc/?region=ap-southeast-1#/elbs/createEnhanceElb)

    - ELB インスタンスを `istio-ingressgateway` サービスにバインド

      ELB インスタンス ID と `loadBalancerIP` を `istio-ingressgateway` に設定します。

{{< text bash >}}
$ kubectl apply -f - <<EOF
apiVersion: v1
kind: Service
metadata:
annotations:
kubernetes.io/elb.class: union
kubernetes.io/elb.id: 4ee43d2b-cec5-4100-89eb-2f77837daa63 # ELB ID
kubernetes.io/elb.lb-algorithm: ROUND_ROBIN
labels:
app: istio-ingressgateway
install.operator.istio.io/owning-resource: unknown
install.operator.istio.io/owning-resource-namespace: istio-system
istio: ingressgateway
istio.io/rev: default
operator.istio.io/component: IngressGateways
operator.istio.io/managed: Reconcile
operator.istio.io/version: 1.9.0
release: istio
name: istio-ingressgateway
namespace: istio-system
spec:
clusterIP: 10.247.7.192
externalTrafficPolicy: Cluster
loadBalancerIP: 119.8.36.132 ## ELB EIP
ports:

- name: status-port
  nodePort: 32484
  port: 15021
  protocol: TCP
  targetPort: 15021
- name: http2
  nodePort: 30294
  port: 80
  protocol: TCP
  targetPort: 8080
- name: https
  nodePort: 31301
  port: 443
  protocol: TCP
  targetPort: 8443
- name: tcp
  nodePort: 30229
  port: 31400
  protocol: TCP
  targetPort: 31400
- name: tls
  nodePort: 32028
  port: 15443
  protocol: TCP
  targetPort: 15443
  selector:
  app: istio-ingressgateway
  istio: ingressgateway
  sessionAffinity: None
  type: LoadBalancer
  EOF
  {{< /text >}}

さまざまな[タスク](/zh/docs/tasks)に挑戦して、Istio の利用を始めてみましょう。
