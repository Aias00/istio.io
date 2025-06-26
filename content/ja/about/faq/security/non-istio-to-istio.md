---
title: 双方向 TLS 認証をグローバルに有効にした場合、非 Istio サービスは Istio サービスにアクセスできますか？
weight: 30
---

`STRICT` 双方向 TLS を有効にすると、非 Istio ワークロードは Istio サービスと通信できません。
これは、有効な Istio クライアント証明書がないためです。

これらのクライアントを許可する場合、双方向 TLS モードを `PERMISSIVE` に設定し、明文と双方向 TLS を許可できます。
これは、単一のワークロードまたはメッシュ全体で行うことができます。

[認証ポリシー](/ja/docs/tasks/security/authentication/authn-policy)を参照してください。
