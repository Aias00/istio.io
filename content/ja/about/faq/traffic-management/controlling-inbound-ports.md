---
title: Sidecar プロキシはどのポートで入站トラフィックをキャプチャしますか？
weight: 20
---

Istio はデフォルトですべてのポートの入站トラフィックをキャプチャします。
`traffic.sidecar.istio.io/includeInboundPorts` Pod 注釈を使用して、キャプチャするポートのグループを指定するか、
`traffic.sidecar.istio.io/excludeOutboundPorts` を使用して、キャプチャを除外するポートのグループを指定することで、
このデフォルトの動作を変更できます。
