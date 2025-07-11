---
title: Zipkin
description: Zipkin との統合方法。
weight: 31
keywords: [integration, zipkin, tracing]
owner: istio/wg-environments-maintainers
test: n/a
---

[Zipkin](https://zipkin.io/) は分散トレーシングシステムで、サービスアーキテクチャの遅延問題の特定に必要なタイミングデータの収集や検索を支援します。

## インストール {#installation}

### 方法 1：クイックスタート {#quick-start}

Istio では Zipkin を素早く起動できる基本的なインストール例を提供しています：

{{< text bash >}}
$ kubectl apply -f {{< github_file >}}/samples/addons/extras/zipkin.yaml
{{< /text >}}

`kubectl apply -f` で Zipkin をクラスタにデプロイします。この例はデモ用であり、パフォーマンスやセキュリティの調整はされていません。

### 方法 2：カスタムインストール {#customizable-install}

[Zipkin ドキュメント](https://zipkin.io/)を参照してインストールを開始してください。Istio で Zipkin を利用する際に特別な変更は不要です。

## 利用方法 {#usage}

Zipkin の利用方法については [Zipkin タスク](/ja/docs/tasks/observability/distributed-tracing/zipkin/) を参照してください。
