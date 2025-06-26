---
title: トラフィックの管理
description: Ambient モードでサービス間のトラフィックを管理します。
weight: 5
owner: istio/wg-networking-maintainers
test: yes
---

waypoint プロキシをインストールしたら、サービス間でトラフィックを分割する方法を学びます。

## 在服务之间分割流量 {#split-traffic-between-services}

Bookinfo アプリケーションには、`reviews` サービスの 3 つのバージョンがあります。
これらのバージョン間でトラフィックを分配して、新機能をテストするか、A/B テストを実行できます。

トラフィックルーティングを設定して、90% のリクエストを `reviews` v1 に、10% のリクエストを `reviews` v2 に送信します：

{{< text syntax=bash snip_id=deploy_httproute >}}
$ kubectl apply -f - <<EOF
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: reviews
spec:
  parentRefs:
  - group: ""
    kind: Service
    name: reviews
    port: 9080
  rules:
  - backendRefs:
    - name: reviews-v1
      port: 9080
      weight: 90
    - name: reviews-v2
      port: 9080
      weight: 10
EOF
{{< /text >}}

100 個のリクエストのトラフィックのうち、約 10% が `reviews-v2` に流れることを確認するには、以下のコマンドを実行してください：

{{< text syntax=bash snip_id=test_traffic_split >}}
$ kubectl exec deploy/curl -- sh -c "for i in \$(seq 1 100); do curl -s http://productpage:9080/productpage | grep reviews-v.-; done"
{{< /text >}}

ほとんどのリクエストは `reviews-v1` に送信されます。ブラウザで Bookinfo アプリケーションを開き、ページを何度も更新すると、これを確認できます。
`reviews-v1` からのリクエストには評価がなく、`reviews-v2` からのリクエストには黒い評価があることに注意してください。

## 下一步 {#next-steps}

このセクションでは、Istio の Ambient モードの入門ガイドをまとめました。
Istio を削除するには、[クリーンアップ](/zh/docs/ambient/getting-started/cleanup)セクションに進んでください。
または、[Ambient モードユーザーガイド](/zh/docs/ambient/usage/)を続けて、Istio の機能と機能についてもっと詳しく学びます。
