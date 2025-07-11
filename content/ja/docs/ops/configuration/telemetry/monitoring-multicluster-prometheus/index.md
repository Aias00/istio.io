---
title: Prometheus を使った Istio マルチクラスター監視
description: Prometheus を使って Istio マルチクラスターを監視する方法。
weight: 10
aliases:
  - /zh/help/ops/telemetry/monitoring-multicluster-prometheus
  - /zh/docs/ops/telemetry/monitoring-multicluster-prometheus
owner: istio/wg-policies-and-telemetry-maintainers
test: no
---

## 概要 {#overview}

このチュートリアルは、2 つ以上の Kubernetes クラスターで構成される Istio メッシュの設定方法を案内します。
これは唯一の方法ではありませんが、Prometheus を使ったマルチクラスターのテレメトリ収集の一例を示します。

Istio マルチクラスター監視には Prometheus の利用を推奨します。主な理由は、Prometheus の[階層型フェデレーション](https://prometheus.io/docs/prometheus/latest/federation/#hierarchical-federation)が利用できるためです。

各クラスターにデプロイされた Istio の Prometheus インスタンスを初期収集器とし、データをメッシュレベルの Prometheus インスタンスに集約します。
メッシュレベルの Prometheus はメッシュ外（外部）にも、メッシュ内のいずれかのクラスターにもデプロイできます。

## Istio マルチクラスターのセットアップ {#multicluster-Istio-setup}

[マルチクラスターインストール](/ja/docs/setup/install/multicluster/)の手順に従い、[マルチクラスター展開モデル](/ja/docs/ops/deployment/deployment-models/#multiple-clusters)から適切なモデルを選択して Istio マルチクラスターを構成してください。
このチュートリアルの目的を達成し、サンプルが動作するように、以下の注意点があります：

**マルチクラスター環境で Istio Prometheus インスタンスを必ず 1 つはインストールしてください！**

各クラスターで独立してデプロイされた Istio Prometheus がクロスクラスター監視の基盤となり、
フェデレーション（Federation）によって本番用 Prometheus インスタンスをメッシュ外またはいずれかのクラスターで稼働させます。

マルチクラスターで稼働している Prometheus インスタンスを確認するには：

{{< text bash >}}
$ kubectl -n istio-system get services prometheus
NAME TYPE CLUSTER-IP EXTERNAL-IP PORT(S) AGE
prometheus ClusterIP 10.8.4.109 <none> 9090/TCP 20h
{{< /text >}}

## Prometheus フェデレーションの設定 {#configure-Prometheus-federation}

### 外部 Prometheus {#external-production-Prometheus}

Istio デプロイメント外で Prometheus インスタンスを稼働させたい理由はいくつか考えられます。
たとえば、長期監視や監視対象クラスターからの分離、複数の独立したメッシュの一元監視などです。
理由はさまざまですが、すべてを機能させるには特別な設定が必要です。

{{< image width="80%"
    link="./external-production-prometheus.svg"
    alt="Istio マルチクラスターを監視する外部 Prometheus のアーキテクチャ"
    caption="Istio マルチクラスターを監視する外部 Prometheus"
    >}}

{{< warning >}}
このチュートリアルではメインクラスターの Prometheus インスタンスへの接続例を示しますが、セキュリティ面は考慮していません。
本番用途では、各 Prometheus エンドポイントへのアクセスを HTTPS で保護してください。
また、パブリックエンドポイントではなく内部ロードバランサを使い、適切なファイアウォールルールを設定するなどの対策を講じてください。
{{< /warning >}}

Istio では [Gateway](/ja/docs/reference/config/networking/gateway/) を使ってクラスターサービスを外部公開できます。
メインクラスターの Prometheus 用に Ingress Gateway を設定し、クラスター内 Prometheus エンドポイントへの外部接続を提供します。

各クラスターについては、[テレメトリプラグインへのリモートアクセス](/ja/docs/tasks/observability/gateways/#option-1-secure-access-https)の手順に従ってください。
また、**必ず**セキュア（HTTPS）アクセスを構成してください。

次に、外部 Prometheus インスタンスを以下のように設定し、メインクラスターの Prometheus インスタンスにアクセスします（Ingress ドメインやクラスター名は適宜置き換えてください）：

{{< text yaml >}}
scrape_configs:

- job_name: 'federate-{{CLUSTER_NAME}}'
  scrape_interval: 15s

  honor_labels: true
  metrics_path: '/federate'

  params:
  'match[]': - '{job="kubernetes-pods"}'

  static_configs: - targets: - 'prometheus.{{INGRESS_DOMAIN}}'
  labels:
  cluster: '{{CLUSTER_NAME}}'
  {{< /text >}}

注意：

- `CLUSTER_NAME` はクラスター作成時の値（`values.global.multiCluster.clusterName` で設定）と一致させてください。

- Prometheus エンドポイント認証は有効化されていません。これにより誰でもメインクラスターの Prometheus にクエリできてしまうため、本番環境では推奨されません。

- Gateway で HTTPS が正しく構成されていない場合、通信はすべて平文となり、これも本番環境では推奨されません。

### クラスター内 Prometheus {#production-Prometheus-on-an-in-mesh-cluster}

いずれかのサブクラスターで Prometheus を稼働させたい場合、メッシュ内の別のメインクラスターの Prometheus インスタンスと接続する必要があります。

これは外部フェデレーション設定のバリエーションです。この場合、メインクラスター上の Prometheus の設定はサブクラスター Prometheus の設定と異なります。

{{< image width="80%"
    link="./in-mesh-production-prometheus.svg"
    alt="Istio マルチクラスターを監視する内部 Prometheus のアーキテクチャ"
    caption="Istio マルチクラスターを監視する内部 Prometheus"
    >}}

Prometheus を設定し、**メイン・サブ**両方のインスタンスにアクセスできるようにします：

まず、以下のコマンドを実行します：

{{< text bash >}}
$ kubectl -n istio-system edit cm prometheus -o yaml
{{< /text >}}

次に、**サブ**クラスター用の設定（各クラスターの Ingress ドメインやクラスター名を置き換え）と、**メイン**クラスター用の設定を追加します：

{{< text yaml >}}
scrape_configs:

- job_name: 'federate-{{REMOTE_CLUSTER_NAME}}'
  scrape_interval: 15s

  honor_labels: true
  metrics_path: '/federate'

  params:
  'match[]': - '{job="kubernetes-pods"}'

  static_configs:

  - targets:
    - 'prometheus.{{REMOTE_INGRESS_DOMAIN}}'
      labels:
      cluster: '{{REMOTE_CLUSTER_NAME}}'

- job_name: 'federate-local'

  honor_labels: true
  metrics_path: '/federate'

  metric_relabel_configs:

  - replacement: '{{CLUSTER_NAME}}'
    target_label: cluster

  kubernetes_sd_configs:

  - role: pod
    namespaces:
    names: ['istio-system']
    params:
    'match[]': - '{**name**=~"istio\_(._)"}' - '{**name**=~"pilot(._)"}'
    {{< /text >}}
