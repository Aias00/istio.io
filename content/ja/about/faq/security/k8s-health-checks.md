---
title: 双方向 TLS 認証を有効にした場合、Kubernetes の liveness と readiness を使用してサービスの正常性を確認するにはどうすればよいですか？
weight: 50
---

双方向 TLS 認証を有効にした場合、kubelet からの HTTP および TCP 正常性チェックは正常に機能しません。
これは、kubelet が Istio が発行した証明書を持たないためです。

有几种选择：

1. probe rewrite を使用して、liveness と readiness のリクエストを直接ワークロードにリダイレクトします。
   [Probe Rewrite](/ja/docs/ops/configuration/mesh/app-health-check/#probe-rewrite) を参照してください。

1. 正常性チェックに別のポートを使用し、通常のサービス ポートでのみ双方向 TLS を有効にします。
   [Istio サービスの正常性チェック](/ja/docs/ops/configuration/mesh/app-health-check/#separate-port) を参照してください。

1. Istio サービスに [`PERMISSIVE` モード](/ja/docs/tasks/security/authentication/mtls-migration)を使用する場合、
   HTTP および双方向 TLS トラフィックを受け入れることができます。
   注意：HTTP トラフィックを使用してこのサービスと通信できるため、双方向 TLS を強制しません。
