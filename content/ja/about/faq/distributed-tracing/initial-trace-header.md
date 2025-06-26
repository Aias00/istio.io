---
title: 最初のトレースヘッダーは誰が生成しますか？
weight: 15
---

リクエストに最初の[ヘッダー](https://www.envoyproxy.io/docs/envoy/latest/configuration/http/http_conn_man/headers#x-request-id)が提供されていない場合、
Istio ゲートウェイまたはサイドカー プロキシ (Envoy) が最初の[ヘッダー](https://www.envoyproxy.io/docs/envoy/latest/configuration/http/http_conn_man/headers#x-request-id)を生成します。
