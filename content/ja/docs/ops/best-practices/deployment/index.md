---
title: Deployment のベストプラクティス
description: Istio サービスメッシュをセットアップする際のベストプラクティス。
force_inline_toc: true
weight: 10
aliases:
  - /ja/docs/ops/prep/deployment
owner: istio/wg-environments-maintainers
test: n/a
---

Istio Deployment を最大限に活用するために、以下の大まかな原則をまとめました。
これらのベストプラクティスは、不適切な設定による影響を最小限に抑え、Deployment 管理を容易にすることを目的としています。

## クラスタは少数大規模でデプロイする {#deploy-fewer-clusters}

Istio は多数の小規模クラスタではなく、少数の大規模クラスタにデプロイするのが望ましいです。
最良の方法は、[名前空間テナンシー](/ja/docs/ops/deployment/deployment-models/#namespace-tenancy)を使って大規模クラスタを管理し、Deployment にクラスタを追加しないことです。この方法により、各リージョンまたはリージョン内の 1 ～ 2 個のクラスタに Istio をデプロイできます。

その後、各リージョンまたはリージョン内の 1 つのクラスタにコントロールプレーンをデプロイし、信頼性を高めます。

## ユーザーの近くにクラスタをデプロイする {#deploy-clusters-near-your-users}

クラスタをグローバルにデプロイし、**エンドユーザーの地理的な近く**で運用します。
これにより、Deployment のレイテンシを低減できます。

## 複数のアベイラビリティゾーンにまたがってデプロイする {#deploy-across-multiple-availability-zones}

Deployment には、各地理的リージョン内で**複数のアベイラビリティゾーン**にまたがるクラスタを含めます。
この方法は、Deployment の{{< gloss "failure domain" >}}障害ドメイン{{< /gloss >}}の範囲を制限し、全体的な障害を回避するのに役立ちます。
