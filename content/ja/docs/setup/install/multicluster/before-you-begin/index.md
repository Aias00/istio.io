---
title: 事前準備
description: 複数クラスタに Istio をインストールする前の初期手順。
weight: 1
icon: setup
keywords: [kubernetes, multicluster]
test: n/a
owner: istio/wg-environments-maintainers
---

複数クラスタのインストールを始める前に、[デプロイメントモデルガイド](/ja/docs/ops/deployment/deployment-models)を確認し、本ガイドで使用される基本的な概念を理解してください。

また、要件を確認し、以下の初期手順を実行してください。

## 要件 {#requirements}

### クラスタ {#cluster}

本ガイドでは、2 つの Kubernetes クラスタが必要です。バージョンは[Kubernetes のサポートバージョン：](/ja/docs/releases/supported-releases#support-status-of-istio-releases){{< supported_kubernetes_versions >}}である必要があります。

### API サーバーアクセス

各クラスタの API サーバーは、メッシュ内の他のクラスタからアクセスできる必要があります。
多くのクラウドプロバイダーは、ネットワークロードバランサー（NLB）を介して API サーバーへのパブリックアクセスを提供しています。
API サーバーに直接アクセスできない場合は、インストール手順を調整してアクセスを許可する必要があります。
たとえば、マルチネットワークやプライマリ・リモート構成で使用される[イーストウエスト](https://en.wikipedia.org/wiki/East-west_traffic)ゲートウェイを利用して、API サーバーへのアクセスを有効にできます。

## 環境変数 {#environment-variables}

本ガイドでは、`cluster1` と `cluster2` の 2 つのクラスタを参照します。
以下の環境変数は、説明を簡略化するために全体を通して使用されます：

| 変数           | 説明                                                                                                                                                                                                      |
| -------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `CTX_CLUSTER1` | [Kubernetes 設定ファイル](https://kubernetes.io/ja/docs/tasks/access-application-cluster/configure-access-multiple-clusters/)のデフォルトコンテキスト名で、クラスタ `cluster1` へのアクセスに使用します。 |
| `CTX_CLUSTER2` | [Kubernetes 設定ファイル](https://kubernetes.io/ja/docs/tasks/access-application-cluster/configure-access-multiple-clusters/)のデフォルトコンテキスト名で、クラスタ `cluster2` へのアクセスに使用します。 |

続行する前に、これら 2 つの変数を設定してください：

{{< text syntax=bash snip_id=none >}}
$ export CTX_CLUSTER1=<your cluster1 context>
$ export CTX_CLUSTER2=<your cluster2 context>
{{< /text >}}

## 信頼関係の構成 {#configure-trust}

マルチクラスタサービスメッシュのデプロイには、メッシュ内のすべてのクラスタ間で信頼関係を確立する必要があります。
システム要件に応じて、信頼関係を確立する方法はいくつかあります。
[証明書管理](/ja/docs/tasks/security/cert-management/)を参照し、利用可能なすべてのオプションの詳細な説明と手順を確認してください。
選択した方法によっては、Istio のインストール手順が若干異なる場合があります。

{{< tip >}}
プライマリクラスタのみをデプロイする予定の場合（ローカル-リモートデプロイメント）、1 つの CA（`cluster1` 上の `istiod`）が 2 つのクラスタに証明書を発行します。
この場合、以下の CA 証明書生成手順をスキップし、デフォルトの自己署名 CA でインストールを行うだけで構いません。
{{< /tip >}}

本ガイドでは、各プライマリクラスタ用に共通ルートから中間証明書を生成することを前提としています。
[手順](/ja/docs/tasks/security/cert-management/plugin-ca-cert/)に従い、CA 証明書と秘密鍵をそれぞれ `cluster1` と `cluster2` に配布してください。

{{< tip >}}
現在、独立した自己署名 CA を持つクラスタがある場合
（[はじめに](/ja/docs/setup/getting-started/)で説明されているような場合）、
[証明書管理](/ja/docs/tasks/security/cert-management/)で紹介されている方法のいずれかを使って CA を変更する必要があります。
CA の変更には通常、Istio の再インストールが必要です。
インストール手順は、選択した CA に応じて変更する必要がある場合があります。
{{< /tip >}}

## 次のステップ {#next-steps}

これで、複数クラスタにまたがる Istio メッシュのインストール準備が整いました。
具体的なインストール手順は、ネットワークやコントロールプレーンのトポロジ要件によって異なります。

ご自身の要件に最適なインストール方法を選択してください：

- [マルチプライマリ構成のインストール](/ja/docs/setup/install/multicluster/multi-primary)

- [プライマリ・リモート構成のインストール](/ja/docs/setup/install/multicluster/primary-remote)

- [異なるネットワーク上でのマルチプライマリ構成のインストール](/ja/docs/setup/install/multicluster/multi-primary_multi-network)

- [異なるネットワーク上でのプライマリ・リモート構成のインストール](/ja/docs/setup/install/multicluster/primary-remote_multi-network)

{{< tip >}}
Helm を使って Istio マルチクラスタをインストールする場合は、まず Helm インストールガイドの[Helm 前提条件](/ja/docs/setup/install/helm/#prerequisites)に従ってください。
{{< /tip >}}

{{< tip >}}
2 つ以上のクラスタにまたがるメッシュの場合、複数のオプションを組み合わせて使用することが必要になる場合があります。
たとえば、各リージョンに 1 つのプライマリクラスタ（マルチプライマリ）、各ゾーンに 1 つのリモートクラスタを配置し、リージョンのプライマリクラスタ（プライマリ・リモート）のコントロールプレーンを使用するなどです。

詳細は[デプロイメントモデル](/ja/docs/ops/deployment/deployment-models)を参照してください。
{{< /tip >}}
