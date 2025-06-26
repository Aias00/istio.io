---
title: Istio のリクエストがどのように流れるかを確認するにはどうすればよいですか？
weight: 80
---

[分散トレース](/ja/docs/tasks/observability/distributed-tracing/)を有効にすることで、
Istio 内のリクエストの流れを確認できます。

また、以下のコマンドを使用して、メッシュ内のより多くの状態情報を確認することもできます：

* [`istioctl proxy-config`](/ja/docs/reference/commands/istioctl/#istioctl-proxy-config)：
  Kubernetes の実行中のプロキシ設定情報を取得します：

    {{< text plain >}}
    # 指定された Pod 内の Envoy インスタンスの起動（bootstrap）設定情報。
    $ istioctl proxy-config bootstrap productpage-v1-bb8d5cbc7-k7qbm

    # 指定された Pod 内の Envoy インスタンスのクラスタ（cluster）設定情報。
    $ istioctl proxy-config cluster productpage-v1-bb8d5cbc7-k7qbm

    # 指定された Pod 内の Envoy インスタンスのリスナー（listener）設定情報。
    $ istioctl proxy-config listener productpage-v1-bb8d5cbc7-k7qbm

    # 指定された Pod 内の Envoy インスタンスのルーティング（route）設定情報。
    $ istioctl proxy-config route productpage-v1-bb8d5cbc7-k7qbm

    # 指定された Pod 内の Envoy インスタンスのエンドポイント（endpoint）設定情報。
    $ istioctl proxy-config endpoints productpage-v1-bb8d5cbc7-k7qbm

    # proxy-config の使用方法の詳細については、以下のコマンドを使用してください。
    $ istioctl proxy-config --help
    {{< /text >}}

* `kubectl get`：ルーティング設定に基づいて、メッシュ内の異なるリソースの情報を取得します：

    {{< text plain >}}
    # すべての VirtualService を一覧表示します。
    $ kubectl get virtualservices
    {{< /text >}}
