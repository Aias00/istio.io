---
title: ローカリティフェイルオーバー
description: このタスクではメッシュにローカリティフェイルオーバーを構成する方法を説明します。
weight: 10
icon: tasks
keywords:
  [locality, load balancing, priority, prioritized, kubernetes, multicluster]
test: yes
owner: istio/wg-networking-maintainers
---

このガイドに従って、メッシュにローカリティフェイルオーバーを構成してください。

始める前に、[始める前に](/ja/docs/tasks/traffic-management/locality-load-balancing/before-you-begin) セクションの手順を完了していることを確認してください。

このタスクでは、`region1.zone1` の `curl` Pod をリクエスト元として `HelloWorld` サービスにリクエストを送信します。
その後、障害を発生させ、以下の順序で異なるローカリティ間でフェイルオーバーが発生することを確認します：

{{< image width="75%"
    link="sequence.svg"
    caption="ローカリティフェイルオーバーの順序"
    >}}

内部的には、[Envoy の優先度](https://www.envoyproxy.io/docs/envoy/latest/intro/arch_overview/upstream/load_balancing/priority.html)がフェイルオーバー制御に使われます。
これらの優先度は、`region1.zone1` の `curl` Pod からのトラフィックに対して次のように割り当てられます：

| 優先度 | ローカリティ    | 詳細                                                                    |
| ------ | --------------- | ----------------------------------------------------------------------- |
| 0      | `region1.zone1` | リージョン・ゾーン・サブゾーンすべて一致                                |
| 1      | なし            | このタスクではサブゾーンを使わないため他のサブゾーン一致はなし          |
| 2      | `region1.zone2` | 同じリージョン内の別ゾーン                                              |
| 3      | `region2.zone3` | 一致なし、ただし `region1`→`region2` のフェイルオーバーが定義されている |
| 4      | `region3.zone4` | 一致なし、`region1`→`region3` のフェイルオーバーは未定義                |

## ローカリティフェイルオーバーの構成 {#configure-locality-failover}

`DestinationRule` を次のように適用します：

- `HelloWorld` サービスに対して[異常検出](/ja/docs/reference/config/networking/destination-rule/#OutlierDetection)を有効にします。
  これはフェイルオーバーが正しく動作するために必須です。
  特に、Sidecar プロキシがサービスのエンドポイントの異常を検知し、次のローカリティへのフェイルオーバーをトリガーできるようにします。
- [フェイルオーバー](/ja/docs/reference/config/networking/destination-rule/#LocalityLoadBalancerSetting-Failover)
  のリージョン間ポリシーを設定します。これにより、リージョンをまたぐフェイルオーバーが予測可能な動作になります。
- [コネクションプール](/ja/docs/reference/config/networking/destination-rule/#ConnectionPoolSettings-http)で、各 HTTP リクエストごとに新しい接続を使うようにします。このタスクでは Envoy の[ドレイン](https://www.envoyproxy.io/docs/envoy/latest/intro/arch_overview/operations/draining)機能を利用して、次のローカリティへのフェイルオーバーを強制します。ドレイン後は Envoy は新規リクエストをすべて拒否します。各リクエストごとに新しい接続を使うことで、ドレイン後すぐにフェイルオーバーが発生します。**この設定はデモ目的のみです。**

{{< text bash >}}
$ kubectl --context="${CTX_PRIMARY}" apply -n sample -f - <<EOF
apiVersion: networking.istio.io/v1
kind: DestinationRule
metadata:
name: helloworld
spec:
host: helloworld.sample.svc.cluster.local
trafficPolicy:
connectionPool:
http:
maxRequestsPerConnection: 1
loadBalancer:
simple: ROUND_ROBIN
localityLbSetting:
enabled: true
failover: - from: region1
to: region2
outlierDetection:
consecutive5xxErrors: 1
interval: 1s
baseEjectionTime: 1m
EOF
{{< /text >}}

## トラフィックが `region1.zone1` に留まることの検証 {#verify-traffic-stays-in-region1zone1}

`curl` Pod から `HelloWorld` サービスを呼び出します：

{{< text bash >}}
$ kubectl exec --context="${CTX_R1_Z1}" -n sample -c curl \
  "$(kubectl get pod --context="${CTX_R1_Z1}" -n sample -l \
 app=curl -o jsonpath='{.items[0].metadata.name}')" \
 -- curl -sSL helloworld.sample:5000/hello
Hello version: region1.zone1, instance: helloworld-region1.zone1-86f77cd7b-cpxhv
{{< /text >}}

レスポンスの `version` が `region1.zone` であることを確認してください。

何度か繰り返し、常に同じレスポンスであることを確認します。

## `region1.zone2` へのフェイルオーバー {#failover-to-region1zone2}

次に、`region1.zone2` へのフェイルオーバーを発生させます。そのために、`region1.zone1` の `HelloWorld` で
[Envoy Sidecar プロキシをドレイン](https://www.envoyproxy.io/docs/envoy/latest/intro/arch_overview/operations/draining#draining)します：

{{< text bash >}}
$ kubectl --context="${CTX_R1_Z1}" exec \
  "$(kubectl get pod --context="${CTX_R1_Z1}" -n sample -l app=helloworld \
 -l version=region1.zone1 -o jsonpath='{.items[0].metadata.name}')" \
 -n sample -c istio-proxy -- curl -sSL -X POST 127.0.0.1:15000/drain_listeners
{{< /text >}}

`curl` Pod から `HelloWorld` サービスを呼び出します：

{{< text bash >}}
$ kubectl exec --context="${CTX_R1_Z1}" -n sample -c curl \
  "$(kubectl get pod --context="${CTX_R1_Z1}" -n sample -l \
 app=curl -o jsonpath='{.items[0].metadata.name}')" \
 -- curl -sSL helloworld.sample:5000/hello
Hello version: region1.zone2, instance: helloworld-region1.zone2-86f77cd7b-cpxhv
{{< /text >}}

最初の呼び出しは失敗し、これがフェイルオーバーをトリガーします。何度か繰り返し、レスポンスの `version` が常に `region1.zone2` であることを確認してください。

## `region2.zone3` へのフェイルオーバー {#failover-to-region2zone3}

次に、`region2.zone3` へのフェイルオーバーを発生させます。先ほどと同様に、`region1.zone2` の `HelloWorld` で呼び出しを失敗させます。

{{< text bash >}}
$ kubectl --context="${CTX_R1_Z2}" exec \
  "$(kubectl get pod --context="${CTX_R1_Z2}" -n sample -l app=helloworld \
 -l version=region1.zone2 -o jsonpath='{.items[0].metadata.name}')" \
 -n sample -c istio-proxy -- curl -sSL -X POST 127.0.0.1:15000/drain_listeners
{{< /text >}}

`curl` Pod から `HelloWorld` サービスを呼び出します：

{{< text bash >}}
$ kubectl exec --context="${CTX_R1_Z1}" -n sample -c curl \
  "$(kubectl get pod --context="${CTX_R1_Z1}" -n sample -l \
 app=curl -o jsonpath='{.items[0].metadata.name}')" \
 -- curl -sSL helloworld.sample:5000/hello
Hello version: region2.zone3, instance: helloworld-region2.zone3-86f77cd7b-cpxhv
{{< /text >}}

最初の呼び出しは失敗し、これがフェイルオーバーをトリガーします。何度か繰り返し、レスポンスの `version` が常に `region2.zone3` であることを確認してください。

## `region3.zone4` へのフェイルオーバー {#failover-to-region3zone4}

次に、`region3.zone4` へのフェイルオーバーを発生させます。先ほどと同様に、`region2.zone3` の `HelloWorld` で呼び出しを失敗させます。

{{< text bash >}}
$ kubectl --context="${CTX_R2_Z3}" exec \
  "$(kubectl get pod --context="${CTX_R2_Z3}" -n sample -l app=helloworld \
 -l version=region2.zone3 -o jsonpath='{.items[0].metadata.name}')" \
 -n sample -c istio-proxy -- curl -sSL -X POST 127.0.0.1:15000/drain_listeners
{{< /text >}}

`curl` Pod から `HelloWorld` サービスを呼び出します：

{{< text bash >}}
$ kubectl exec --context="${CTX_R1_Z1}" -n sample -c curl \
  "$(kubectl get pod --context="${CTX_R1_Z1}" -n sample -l \
 app=curl -o jsonpath='{.items[0].metadata.name}')" \
 -- curl -sSL helloworld.sample:5000/hello
Hello version: region3.zone4, instance: helloworld-region3.zone4-86f77cd7b-cpxhv
{{< /text >}}

最初の呼び出しは失敗し、これがフェイルオーバーをトリガーします。何度か繰り返し、レスポンスの `version` が常に `region3.zone4` であることを確認してください。

**おめでとうございます！** ローカリティフェイルオーバーの構成に成功しました！

## 次のステップ {#next-steps}

[クリーンアップ](/ja/docs/tasks/traffic-management/locality-load-balancing/cleanup) タスクでリソースやファイルを削除してください。
