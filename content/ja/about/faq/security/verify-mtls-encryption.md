---
title: トラフィックが双方向 TLS 暗号化を使用しているかどうかを確認するにはどうすればよいですか？
weight: 25
---

`values.global.proxy.privileged=true` パラメーターを使用して Istio をインストールした場合、
`tcpdump` を使用して暗号化状態を確認できます。同様に、Kubernetes 1.23 以降のバージョンでは、
Istio を特権ユーザーとしてインストールする別の選択肢として、`kubectl debug`
を使用して、[一時コンテナ（Ephemeral Container）](https://kubernetes.io/ja/docs/tasks/debug/debug-application/debug-running-pod/#ephemeral-container)
で `tcpdump` を実行できます。[Istio 双方向 TLS 移行](/ja/docs/tasks/security/authentication/mtls-migration)を参照してください。
