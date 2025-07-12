---
title: はじめに
description: 快速、轻松地尝试 Istio 特性。
weight: 5
aliases:
  - /zh/docs/setup/additional-setup/getting-started/
  - /zh/latest/docs/setup/additional-setup/getting-started/
keywords:
  [getting-started, install, bookinfo, quick-start, kubernetes, gateway-api]
test: yes
owner: istio/wg-environments-maintainers
---

{{< tip >}}
Istio の {{< gloss "ambient" >}}Ambient モード{{< /gloss >}}を試したいですか？
[Ambient モード入門](/ja/docs/ambient/getting-started)ガイドをご覧ください！
{{< /tip >}}

このガイドでは、Istio を素早く評価できます。すでに Istio に慣れている場合や、他の構成タイプや高度な[デプロイメントモデル](/ja/docs/ops/deployment/deployment-models/)に興味がある場合は、[どの Istio インストール方法を使うべきか？](/ja/about/faq/#install-method-selection)FAQ ページをご参照ください。

Kubernetes クラスターが必要です。クラスターをお持ちでない場合は、[kind](/ja/docs/setup/platform-setup/kind) やその他の[サポートされている Kubernetes プラットフォーム](/ja/docs/setup/platform-setup)を利用できます。

以下の手順で Istio を始めましょう：

1. [Istio をダウンロードしてインストール](#download)
1. [Kubernetes Gateway API CRD をインストール](#gateway-api)
1. [サンプルアプリケーションをデプロイ](#bookinfo)
1. [アプリケーションを外部公開](#ip)
1. [ダッシュボードを確認](#dashboard)

## Istio のダウンロード {#download}

1. [Istio リリース]({{< istio_release_url >}})ページにアクセスし、ご利用の OS に合ったインストーラをダウンロードするか、[最新バージョンを自動ダウンロード](/ja/docs/setup/additional-setup/download-istio-release)（Linux または macOS）：

   {{< text bash >}}
    $ curl -L https://istio.io/downloadIstio | sh -
   {{< /text >}}

1. Istio パッケージディレクトリに移動します。たとえば、パッケージが `istio-{{< istio_full_version >}}` の場合：

   {{< text syntax=bash snip_id=none >}}
    $ cd istio-{{< istio_full_version >}}
   {{< /text >}}

   インストールディレクトリには以下が含まれます：

   - `samples/` ディレクトリのサンプルアプリケーション
   - `bin/` ディレクトリの [`istioctl`](/ja/docs/reference/commands/istioctl) クライアントバイナリ

1. `istioctl` クライアントをパスに追加します（Linux または macOS）：

   {{< text bash >}}
    $ export PATH=$PWD/bin:$PATH
   {{< /text >}}

## Istio のインストール {#install}

このガイドでは `demo` [プロファイル](/ja/docs/setup/additional-setup/config-profiles/)を使用します。これはテストに適したデフォルト設定を持つためですが、他にも本番やパフォーマンステスト、[OpenShift](/ja/docs/setup/platform-setup/openshift/)向けのプロファイルもあります。

[Istio Gateway](/ja/docs/concepts/traffic-management/#gateways)とは異なり、[Kubernetes Gateway](https://gateway-api.sigs.k8s.io/api-types/gateway/)を作成すると、デフォルトで[ゲートウェイプロキシサーバー](/ja/docs/tasks/traffic-management/ingress/gateway-api/#automated-deployment)もデプロイされます。今回はそれらを使わないため、通常 `demo` プロファイルでインストールされるデフォルトの Istio Gateway サービスのデプロイを無効化します。

1. `demo` プロファイルで Istio をインストールし、Gateway を無効化：

   {{< text bash >}}
    $ istioctl install -f @samples/bookinfo/demo-profile-no-gateways.yaml@ -y
    ✔ Istio core installed
    ✔ Istiod installed
    ✔ Installation complete
    Made this installation the default for injection and validation.
   {{< /text >}}

1. 名前空間にラベルを付与し、アプリケーションデプロイ時に自動で Envoy Sidecar プロキシが注入されるようにします：

   {{< text bash >}}
    $ kubectl label namespace default istio-injection=enabled
    namespace/default labeled
   {{< /text >}}

## Kubernetes Gateway API CRD のインストール {#gateway-api}

Kubernetes Gateway API CRD は多くの Kubernetes クラスターでデフォルトではインストールされていません。Gateway API を使う前に必ずインストールしてください。

1. Gateway API CRD がまだなければ、インストールします：

   {{< text bash >}}
    $ kubectl get crd gateways.gateway.networking.k8s.io &> /dev/null || \
    { kubectl kustomize "github.com/kubernetes-sigs/gateway-api/config/crd?ref={{< k8s_gateway_api_version >}}" | kubectl apply -f -; }
   {{< /text >}}

## サンプルアプリケーションのデプロイ {#bookinfo}

Istio を設定したので、`default` 名前空間にデプロイするアプリケーションには Sidecar コンテナが自動注入されます。

1. [`Bookinfo` サンプルアプリ](/ja/docs/examples/bookinfo/)をデプロイ：

   {{< text bash >}}
    $ kubectl apply -f @samples/bookinfo/platform/kube/bookinfo.yaml@
    service/details created
    serviceaccount/bookinfo-details created
    deployment.apps/details-v1 created
    service/ratings created
    serviceaccount/bookinfo-ratings created
    deployment.apps/ratings-v1 created
    service/reviews created
    serviceaccount/bookinfo-reviews created
    deployment.apps/reviews-v1 created
    deployment.apps/reviews-v2 created
    deployment.apps/reviews-v3 created
    service/productpage created
    serviceaccount/bookinfo-productpage created
    deployment.apps/productpage-v1 created
   {{< /text >}}

   アプリケーションはすぐに起動します。各 Pod が Ready になると、Istio Sidecar も一緒にデプロイされます。

   {{< text bash >}}
    $ kubectl get services
    NAME TYPE CLUSTER-IP EXTERNAL-IP PORT(S) AGE
    details ClusterIP 10.0.0.212 <none> 9080/TCP 29s
    kubernetes ClusterIP 10.0.0.1 <none> 443/TCP 25m
    productpage ClusterIP 10.0.0.57 <none> 9080/TCP 28s
    ratings ClusterIP 10.0.0.33 <none> 9080/TCP 29s
    reviews ClusterIP 10.0.0.28 <none> 9080/TCP 29s
   {{< /text >}}

   および

   {{< text bash >}}
    $ kubectl get pods
    NAME READY STATUS RESTARTS AGE
    details-v1-558b8b4b76-2llld 2/2 Running 0 2m41s
    productpage-v1-6987489c74-lpkgl 2/2 Running 0 2m40s
    ratings-v1-7dc98c7588-vzftc 2/2 Running 0 2m41s
    reviews-v1-7f99cc4496-gdxfn 2/2 Running 0 2m41s
    reviews-v2-7d79d5bd5d-8zzqd 2/2 Running 0 2m41s
    reviews-v3-7dbcdcbc56-m8dph 2/2 Running 0 2m41s
   {{< /text >}}

   Pod の `READY 2/2` 表示で、アプリケーションコンテナと Istio Sidecar コンテナが両方稼働していることが確認できます。

1. レスポンスのページタイトルを確認して、アプリケーションがクラスタ内で動作していることを検証します：

   {{< text bash >}}
    $ kubectl exec "$(kubectl get pod -l app=ratings -o jsonpath='{.items[0].metadata.name}')" -c ratings -- curl -sS productpage:9080/productpage | grep -o "<title>.\*</title>"
    <title>Simple Bookstore App</title>
   {{< /text >}}

## アプリケーションの外部公開 {#ip}

Bookinfo アプリケーションはデプロイされましたが、外部からはアクセスできません。アクセス可能にするには、Ingress Gateway を作成し、パスをメッシュのエッジルーターにマッピングする必要があります。

1. Bookinfo アプリ用の [Kubernetes Gateway](https://gateway-api.sigs.k8s.io/api-types/gateway/) を作成：

   {{< text syntax=bash snip_id=deploy_bookinfo_gateway >}}
    $ kubectl apply -f @samples/bookinfo/gateway-api/bookinfo-gateway.yaml@
    gateway.gateway.networking.k8s.io/bookinfo-gateway created
    httproute.gateway.networking.k8s.io/bookinfo created
   {{< /text >}}

   デフォルトで、Istio は Gateway 用に `LoadBalancer` サービスを作成します。
   今回はトンネル経由でアクセスするためロードバランサーは不要です。
   外部 IP アドレスでのロードバランサー設定方法は[Ingress Gateway](/ja/docs/tasks/traffic-management/ingress/ingress-control/)ドキュメントをご覧ください。

1. アノテーションで Gateway のサービス種別を `ClusterIP` に変更：

   {{< text syntax=bash snip_id=annotate_bookinfo_gateway >}}
    $ kubectl annotate gateway bookinfo-gateway networking.istio.io/service-type=ClusterIP --namespace=default
   {{< /text >}}

1. Gateway の状態を確認するには：

   {{< text bash >}}
    $ kubectl get gateway
    NAME CLASS ADDRESS PROGRAMMED AGE
    bookinfo-gateway istio bookinfo-gateway-istio.default.svc.cluster.local True 42s
   {{< /text >}}

## アプリケーションへのアクセス {#access-the-application}

先ほど設定した Gateway 経由で Bookinfo の `productpage` サービスに接続します。
Gateway へアクセスするには `kubectl port-forward` コマンドを使います：

{{< text syntax=bash snip_id=none >}}
$ kubectl port-forward svc/bookinfo-gateway-istio 8080:80
{{< /text >}}

ブラウザで `http://localhost:8080/productpage` にアクセスし、Bookinfo アプリケーションを表示してください。

{{< image width="80%" link="./bookinfo-browser.png" caption="Bookinfo 应用程序" >}}

如果您刷新页面，您应该会看到书评和评分发生变化，
因为请求分布在 `reviews` 服务的不同版本上。

## 查看仪表板 {#dashboard}

Istio 和[几个遥测应用](/ja/docs/ops/integrations)做了集成。
遥测能帮您了解服务网格的结构、展示网络的拓扑结构、分析网格的健康状态。

使用下面说明部署 [Kiali](/ja/docs/ops/integrations/kiali/) 仪表板、
以及 [Prometheus](/ja/docs/ops/integrations/prometheus/)、
[Grafana](/ja/docs/ops/integrations/grafana)、
还有 [Jaeger](/ja/docs/ops/integrations/jaeger/)。

1.  安装 [Kiali 和其他插件]({{< github_tree >}}/samples/addons)，等待部署完成。

    {{< text bash >}}
    $ kubectl apply -f @samples/addons@
    $ kubectl rollout status deployment/kiali -n istio-system
    Waiting for deployment "kiali" rollout to finish: 0 of 1 updated replicas are available...
    deployment "kiali" successfully rolled out
    {{< /text >}}

1.  访问 Kiali 仪表板。

    {{< text bash >}}
    $ istioctl dashboard kiali
    {{< /text >}}

1.  在左侧的导航菜单，选择 **Graph**，
    然后在 **Namespace** 下拉列表中，选择 **default**。

    {{< tip >}}
    {{< boilerplate trace-generation >}}
    {{< /tip >}}

    Kiali 仪表板展示了网格的概览以及 `Bookinfo` 示例应用的各个服务之间的关系。
    它还提供过滤器来可视化流量的流动。

    {{< image link="./kiali-example2.png" caption="Kiali 仪表板" >}}

## 后续步骤 {#next-steps}

恭喜您完成了评估安装！

对于新手来说，以下这些任务是非常好的学习资源，
可以借助 `demo` 安装更深入评估 Istio 的特性：

- [请求路由](/ja/docs/tasks/traffic-management/request-routing/)
- [错误注入](/ja/docs/tasks/traffic-management/fault-injection/)
- [流量切换](/ja/docs/tasks/traffic-management/traffic-shifting/)
- [查询指标](/ja/docs/tasks/observability/metrics/querying-metrics/)
- [可视化指标](/ja/docs/tasks/observability/metrics/using-istio-dashboard/)
- [访问外部服务](/ja/docs/tasks/traffic-management/egress/egress-control/)
- [可视化网格](/ja/docs/tasks/observability/kiali/)

在您为生产系统定制 Istio 之前，请先参阅这些学习资源：

- [部署模型](/ja/docs/ops/deployment/deployment-models/)
- [部署的最佳实践](/ja/docs/ops/best-practices/deployment/)
- [Pod 的要求](/ja/docs/ops/deployment/application-requirements/)
- [通用安装说明](/ja/docs/setup/)

## 加入 Istio 社区 {#join-the-istio-community}

我们欢迎您加入 [Istio 社区](/ja/get-involved/)，
提出问题，并给我们以反馈。

## 卸载 {#uninstall}

要删除 `Bookinfo` 示例应用和配置，请参阅[清理 `Bookinfo`](/ja/docs/examples/bookinfo/#cleanup)。

Istio 卸载程序按照层次结构逐级地从 `istio-system`
命令空间中删除 RBAC 权限和所有资源。对于不存在的资源报错，
可以安全地忽略掉，毕竟它们已经被分层地删除了。

{{< text bash >}}
$ kubectl delete -f @samples/addons@
$ istioctl uninstall -y --purge
{{< /text >}}

命名空间 `istio-system` 默认情况下并不会被移除。
不需要的时候，使用下面命令移除它：

{{< text bash >}}
$ kubectl delete namespace istio-system
{{< /text >}}

指示 Istio 自动注入 Envoy Sidecar 代理的标签默认也不移除。
不需要的时候，使用下面命令移除它。

{{< text bash >}}
$ kubectl label namespace default istio-injection-
{{< /text >}}

如果您安装了 Kubernetes Gateway API CRD 并且现在想要删除它们，请运行以下命令之一：

- 如果您运行的任何任务需要**实验版本**的 CRD：

  {{< text bash >}}
    $ kubectl kustomize "github.com/kubernetes-sigs/gateway-api/config/crd/experimental?ref={{< k8s_gateway_api_version >}}" | kubectl delete -f -
  {{< /text >}}

- 否则：

  {{< text bash >}}
    $ kubectl kustomize "github.com/kubernetes-sigs/gateway-api/config/crd?ref={{< k8s_gateway_api_version >}}" | kubectl delete -f -
  {{< /text >}}
