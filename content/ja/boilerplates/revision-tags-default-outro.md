---
---
`default` ラベルと既存の未リビジョンの Istio インストール方式を使用する場合、
古い `MutatingWebhookConfiguration`（通常は `istio-sidecar-injector` と呼ばれます）を削除することをお勧めします。
これにより、新旧のコントロールプレーンが同時に注入を試みるのを防ぎます。
