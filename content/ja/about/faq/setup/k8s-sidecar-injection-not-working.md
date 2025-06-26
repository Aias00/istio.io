---
title: Kubernetes - Sidecar 自動注入の問題をデバッグするにはどうすればよいですか？
weight: 20
---

Sidecar 自動注入をサポートするには、クラスターがこの
[前提条件](/ja/docs/setup/additional-setup/sidecar-injection/#automatic-sidecar-injection)を満たしていることを確認してください。
もしマイクロサービスが `kube-system`、`kube-public`、または `istio-system` という名前空間にデプロイされている場合、
Sidecar 自動注入は免除されます。代わりに、他の名前空間を使用してください。
