---
title: Jaeger
description: Jaeger との統合方法。
weight: 28
keywords: [integration, jaeger, tracing]
owner: istio/wg-environments-maintainers
test: n/a
---

[Jaeger](https://www.jaegertracing.io/) はオープンソースのエンドツーエンド分散トレーシングシステムで、複雑な分散システムの監視や障害解析を可能にします。

## インストール {#installation}

### 方法 1：クイックスタート {#option-1-quick-start}

Istio では Jaeger を素早く起動できる基本的なインストール例を提供しています：

{{< text bash >}}
$ kubectl apply -f {{< github_file >}}/samples/addons/jaeger.yaml
{{< /text >}}

このコマンドで Jaeger がクラスタにデプロイされます。これは例示用であり、パフォーマンスやセキュリティの調整はされていません。

### 方法 2：手動インストール {#option-2-customizable-install}

[Jaeger ドキュメント](https://www.jaegertracing.io/)を参照してインストールを開始してください。Istio で Jaeger を使う際に特別な設定は不要です。

## 利用方法 {#usage}

Jaeger の利用方法については [Jaeger タスク](/ja/docs/tasks/observability/distributed-tracing/jaeger/) を参照してください。
