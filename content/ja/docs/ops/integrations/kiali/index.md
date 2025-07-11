---
title: Kiali
description: Kiali との統合方法に関する情報。
weight: 29
keywords: [integration, kiali]
owner: istio/wg-environments-maintainers
test: no
---

[Kiali](https://kiali.io/) はサービスメッシュの設定や検証機能を備えた Istio の可観測性コンソールです。トラフィックを監視してトポロジやエラーを可視化し、サービスメッシュの構造や状態を把握できます。
Kiali は詳細な指標を提供し、Grafana との基本的な統合も可能で、高度なクエリにも対応します。また、[Jaeger](/ja/docs/ops/integrations/jaeger) と連携して分散トレーシング機能も提供します。

## インストール {#installation}

### 方法 1：クイックスタート {#option-1-quick-start}

Istio では Kiali を素早く起動できる基本的なインストール例を提供しています：

{{< text bash >}}
$ kubectl apply -f {{< github_file >}}/samples/addons/kiali.yaml
{{< /text >}}

このコマンドで Kiali がクラスタにデプロイされます。これはデモ用であり、パフォーマンスやセキュリティの調整はされていません。

{{< idea >}}
このサンプル YAML を使って Kiali を外部公開する場合、`anonymous` 以外の認証ポリシーを使う際は ConfigMap の `signing_key` を必ず変更してください。
{{< /idea >}}

### 方法 2：カスタムインストール {#option-2-customizable-install}

Kiali プロジェクトは[クイックスタートガイド](https://kiali.io/docs/installation/quick-start)や[カスタムインストール方法](https://kiali.io/docs/installation/installation-guide)を提供しています。運用環境ではこれらの手順に従い、最新バージョンやベストプラクティスを確認してください。

## 利用方法 {#usage}

Kiali の利用方法については[メッシュの可視化](/ja/docs/tasks/observability/kiali/)タスクを参照してください。
