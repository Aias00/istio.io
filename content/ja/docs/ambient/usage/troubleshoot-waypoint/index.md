---
title: waypoint のトラブルシューティング
description: waypoint プロキシ経由のルーティング問題の解決方法。
weight: 70
owner: istio/wg-networking-maintainers
test: no
---

本ガイドでは、ネームスペース・サービス・ワークロードを waypoint プロキシに登録したにもかかわらず、期待通りの動作が得られない場合の対処方法を説明します。

## トラフィックルーティングやセキュリティポリシーの問題 {#problems-with-traffic-routing-or-security-policy}

`productpage` サービス経由で `curl` Pod から `reviews` サービスにリクエストを送信します：

{{< text bash >}}
$ kubectl exec deploy/curl -- curl -s http://productpage:9080/productpage
{{< /text >}}

`curl` Pod から `reviews` の `v2` Pod へリクエストを送信します：

{{< text bash >}}
$ export REVIEWS_V2_POD_IP=$(kubectl get pod -l version=v2,app=reviews -o jsonpath='{.items[0].status.podIP}')
$ kubectl exec deploy/curl -- curl -s http://$REVIEWS_V2_POD_IP:9080/reviews/1
{{< /text >}}

`reviews` サービスへのリクエストは `reviews-svc-waypoint` で全ての L7 ポリシーが強制されるべきです。
`reviews` の `v2` Pod へのリクエストは `reviews-v2-pod-waypoint` で全ての L7 ポリシーが強制されるべきです。

1. L7 設定が適用されていない場合、まず `istioctl analyze` を実行して設定に検証エラーがないか確認してください。

   {{< text bash >}}
   $ istioctl analyze
   ✔ No validation issues found when analyzing namespace: default.
   {{< /text >}}

1. どの waypoint がサービスや Pod に L7 設定を適用しているかを特定します。

   ソースがサービスのホスト名や IP でターゲットを呼び出している場合、
   `istioctl experimental ztunnel-config service` コマンドでターゲットサービスがどの waypoint を使っているか確認できます。
   先ほどの例では、`reviews` サービスは `reviews-svc-waypoint` を使い、
   `default` ネームスペースの他のサービスは `waypoint` ネームスペースを使うはずです。

   {{< text bash >}}
   $ istioctl ztunnel-config service
   NAMESPACE SERVICE NAME SERVICE VIP WAYPOINT
   default bookinfo-gateway-istio 10.43.164.194 waypoint
   default bookinfo-gateway-istio 10.43.164.194 waypoint
   default bookinfo-gateway-istio 10.43.164.194 waypoint
   default bookinfo-gateway-istio 10.43.164.194 waypoint
   default details 10.43.160.119 waypoint
   default kubernetes 10.43.0.1 waypoint
   default productpage 10.43.172.254 waypoint
   default ratings 10.43.71.236 waypoint
   default reviews 10.43.162.105 reviews-svc-waypoint
   ...
   {{< /text >}}

   ソースが Pod IP でターゲットを呼び出している場合、`istioctl ztunnel-config workload` コマンドでターゲット Pod がどの waypoint を使っているか確認できます。
   先ほどの例では、`reviews` の `v2` Pod は `reviews-v2-pod-waypoint` を使い、
   `default` ネームスペースの他の Pod には waypoint はありません。
   （デフォルトでは [waypoint はサービス向けトラフィックのみ処理します](/ja/docs/ambient/usage/waypoint/#waypoint-traffic-types)）。

   {{< text bash >}}
   $ istioctl ztunnel-config workload
   NAMESPACE POD NAME IP NODE WAYPOINT PROTOCOL
   default bookinfo-gateway-istio-7c57fc4647-wjqvm 10.42.2.8 k3d-k3s-default-server-0 None TCP
   default details-v1-698d88b-wwsnv 10.42.2.4 k3d-k3s-default-server-0 None HBONE
   default productpage-v1-675fc69cf-fp65z 10.42.2.6 k3d-k3s-default-server-0 None HBONE
   default ratings-v1-6484c4d9bb-crjtt 10.42.0.4 k3d-k3s-default-agent-0 None HBONE
   default reviews-svc-waypoint-c49f9f569-b492t 10.42.2.10 k3d-k3s-default-server-0 None TCP
   default reviews-v1-5b5d6494f4-nrvfx 10.42.2.5 k3d-k3s-default-server-0 None HBONE
   default reviews-v2-5b667bcbf8-gj7nz 10.42.0.5 k3d-k3s-default-agent-0 reviews-v2-pod-waypoint HBONE
   ...
   {{< /text >}}

   Pod の waypoint 列の値が正しくない場合、Pod に `istio.io/use-waypoint` ラベルが付与されているか、
   その値がワークロードトラフィックを処理できる waypoint 名になっているか確認してください。
   例えば、`reviews` の `v2` Pod が使う waypoint がサービストラフィックのみ処理可能な場合、
   その Pod には waypoint が割り当てられません。
   Pod の `istio.io/use-waypoint` ラベルが正しそうな場合、waypoint の Gateway リソースに `istio.io/waypoint-for` が適切な値（Pod なら `all` または `workload`）で付与されているか確認してください。

1. `istioctl proxy-status` コマンドで waypoint のプロキシ状態を確認します。

   {{< text bash >}}
   $ istioctl proxy-status
   NAME CLUSTER CDS LDS EDS RDS ECDS ISTIOD VERSION
   bookinfo-gateway-istio-7c57fc4647-wjqvm.default Kubernetes SYNCED SYNCED SYNCED SYNCED NOT SENT istiod-795d55fc6d-vqtjx 1.23-alpha.75c6eafc5bc8d160b5643c3ea18acb9785855564
   reviews-svc-waypoint-c49f9f569-b492t.default Kubernetes SYNCED SYNCED SYNCED NOT SENT NOT SENT istiod-795d55fc6d-vqtjx 1.23-alpha.75c6eafc5bc8d160b5643c3ea18acb9785855564
   reviews-v2-pod-waypoint-7f5dbd597-7zzw7.default Kubernetes SYNCED SYNCED NOT SENT NOT SENT NOT SENT istiod-795d55fc6d-vqtjx 1.23-alpha.75c6eafc5bc8d160b5643c3ea18acb9785855564
   waypoint-6f7b665c89-6hppr.default Kubernetes SYNCED SYNCED SYNCED NOT SENT NOT SENT istiod-795d55fc6d-vqtjx 1.23-alpha.75c6eafc5bc8d160b5643c3ea18acb9785855564
   ...
   {{< /text >}}

1. Envoy の[アクセスログ](/ja/docs/tasks/observability/logs/access-log/)を有効化し、リクエスト送信後に waypoint プロキシのログを確認します：

   {{< text bash >}}
   $ kubectl logs deploy/waypoint
   {{< /text >}}

   さらに情報が必要な場合は、waypoint プロキシでデバッグログを有効化できます：

   {{< text bash >}}
   $ istioctl pc log deploy/waypoint --level debug
   {{< /text >}}

1. `istioctl proxy-config` コマンドで waypoint の Envoy 設定を確認します。
   このコマンドは waypoint に関連するクラスタ・エンドポイント・リスナー・ルート・証明書などの情報を表示します：

   {{< text bash >}}
   $ istioctl proxy-config all deploy/waypoint
   {{< /text >}}

Envoy のデバッグ方法については、[Envoy 設定の詳細調査](/ja/docs/ops/diagnostic-tools/proxy-cmd/#deep-dive-into-envoy-configuration)も参照してください。waypoint プロキシは Envoy ベースです。
