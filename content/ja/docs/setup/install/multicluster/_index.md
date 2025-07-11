---
title: マルチクラスタインストール
description: 複数の Kubernetes クラスタにまたがって Istio サービスメッシュをインストールします。
weight: 40
aliases:
  - /zh/docs/setup/kubernetes/multicluster-install/
  - /zh/docs/setup/kubernetes/multicluster/
  - /zh/docs/setup/kubernetes/install/multicluster/
  - /zh/docs/setup/install/multicluster/gateways/
  - /zh/docs/setup/install/multicluster/shared/
keywords: [kubernetes, multicluster]
simple_list: true
content_above: true
test: table-of-contents
owner: istio/wg-environments-maintainers
---

このガイドに従って、複数の{{< gloss "cluster" >}}クラスタ{{< /gloss >}}にまたがる
Istio {{< gloss "service mesh" >}}サービスメッシュ{{< /gloss >}}をインストールします。

このガイドでは、{{< gloss "multicluster" >}}マルチクラスタ{{< /gloss >}}メッシュを作成する際によくある課題について説明します。

- [ネットワークトポロジー](/zh/docs/ops/deployment/deployment-models#network-models)：
  1 つまたは 2 つのネットワーク

- [コントロールプレーントポロジー](/zh/docs/ops/deployment/deployment-models#control-plane-models)：
  複数の{{< gloss "primary cluster" >}}プライマリクラスタ{{< /gloss >}}、
  プライマリ{{< gloss "remote cluster" >}}リモートクラスタ{{< /gloss >}}

{{< tip >}}
2 つ以上のクラスタにまたがるメッシュの場合は、このガイドの手順を拡張して、より複雑なトポロジーを構成できます。

詳細は[デプロイメントモデル](/zh/docs/ops/deployment/deployment-models)を参照してください。
{{< /tip >}}
