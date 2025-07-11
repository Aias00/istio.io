---
title: Apache SkyWalking
description: Apache SkyWalking との統合方法。
weight: 32
keywords: [integration, skywalking, tracing]
owner: istio/wg-environments-maintainers
test: no
---

[Apache SkyWalking](http://skywalking.apache.org) は、マイクロサービス、クラウドネイティブ、コンテナなどのアーキテクチャ向けに設計されたアプリケーションパフォーマンス監視（APM）システムです。SkyWalking は可観測性のワンストップソリューションであり、Jaeger や Zipkin のような分散トレーシング、Prometheus や Grafana のようなメトリクス、Kiali のようなログ記録機能を備えています。さらに、ログとトレースの関連付け、システムイベントの収集とメトリクスとの関連付け、eBPF ベースのサービスパフォーマンス分析など、多くのシナリオに可観測性を拡張できます。

## インストール {#installation}

### オプション 1：クイックスタート {#option-1-quick-start}

Istio では SkyWalking を素早く起動できる基本的なインストール例を提供しています：

{{< text bash >}}
$ kubectl apply -f @samples/addons/extras/skywalking.yaml@
{{< /text >}}

このコマンドで SkyWalking がクラスタにデプロイされます。この例はデモ用であり、パフォーマンスやセキュリティの調整はされていません。

Istio プロキシはデフォルトで SkyWalking にトレースを送信しません。SkyWalking トレース拡張プロバイダーを有効にするには、以下のフィールドを設定してください：

{{< text yaml >}}
apiVersion: install.istio.io/v1alpha1
kind: IstioOperator
spec:
meshConfig:
extensionProviders: - skywalking:
service: tracing.istio-system.svc.cluster.local
port: 11800
name: skywalking
defaultProviders:
tracing: - "skywalking"
{{< /text >}}

### オプション 2：カスタムインストール {#option-2-customizable-install}

[SkyWalking ドキュメント](http://skywalking.apache.org)を参照してインストールを開始してください。Istio で SkyWalking を利用する際に特別な変更は不要です。

インストール後は、`skywalking-oap` Deployment を指す `--set meshConfig.extensionProviders[0].skywalking.service` オプションを修正してください。TLS 設定などの高度な設定は [`ProxyConfig.Tracing`](/ja/docs/reference/config/istio.mesh.v1alpha1/#Tracing) を参照してください。

## 利用方法 {#usage}

SkyWalking の利用方法については [SkyWalking タスク](/ja/docs/tasks/observability/distributed-tracing/skywalking/) を参照してください。
