---
title: Auto 双方向 TLS は "excludeInboundPorts" 注釈で設定されたポートを除外しますか？
weight: 80
---

いいえ、`traffic.sidecar.istio.io/excludeInboundPorts` がサーバー ワークロードで使用されている場合、
Istio は依然としてクライアント Envoy をデフォルトで構成し、双方向 TLS リクエストを送信します。
これを変更するには、双方向 TLS モードを `DISABLE` に設定するターゲット ルールを構成する必要があります。
これにより、クライアントはこれらのポートに純粋なテキストを送信します。
