---
---
{{< warning >}}

デフォルトの Chart 設定は、Istio プロキシが使用するサービスアカウントトークンの投影として安全なサードパーティのトークンを使用し、
Istio コントロールプレーンに対して認証を行います。
以下の Chart をインストールする前に、[ここ](/ja/docs/ops/best-practices/security/#configure-third-party-service-account-tokens) に記載されている手順に従って、サードパーティのトークンが有効になっていることを確認してください。
サードパーティのトークンが有効になっていない場合、`--set global.jwtPolicy=first-party-jwt` を Helm のインストールコマンドに追加してください。
`jwtPolicy` が正しく設定されていない場合、`istiod` は `istio-token` ボリュームが不足しているため、ゲートウェイまたは Envoy プロキシが注入されたワークロードに関連付けられた Pod はデプロイされません。
{{< /warning >}}
