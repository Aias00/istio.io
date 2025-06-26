---
---
## 先决条件 {#prerequisites}

1. 必要な[特定のプラットフォームの設定](/ja/docs/setup/platform-setup/)を実行します。

1. [Pod とサービスの要件](/ja/docs/ops/deployment/application-requirements/)を確認します。

1. [最新の Helm クライアントをインストール](https://helm.sh/docs/intro/install/)します。
   現在サポートされている最古の Istio バージョンより前にリリースされた Helm バージョンは、テスト、サポート、または推奨されていません。

1. Helm リポジトリを設定します：

{{< text bash >}}
$ helm repo add istio https://istio-release.storage.googleapis.com/charts
$ helm repo update
{{< /text >}}
