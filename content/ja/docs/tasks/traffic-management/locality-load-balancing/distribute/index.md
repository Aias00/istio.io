---
title: ローカリティ重み付き分散
description: このガイドではローカリティ重み付き分散の構成方法を説明します。
weight: 20
icon: tasks
keywords: [locality, load balancing, kubernetes, multicluster]
test: yes
owner: istio/wg-networking-maintainers
---

このガイドに従って、リージョン間のトラフィック分散を構成します。

続行する前に、[始める前に](/ja/docs/tasks/traffic-management/locality-load-balancing/before-you-begin) セクションの手順を完了していることを確認してください。

このタスクでは、`region1` `zone1` の `curl` Pod を `HelloWorld` サービスへのリクエスト元として使用します。
以下のように異なるローカリティに分散するように Istio を構成します：

| リージョン | ゾーン  | トラフィック(%) |
| ---------- | ------- | --------------- |
| `region1`  | `zone1` | 70              |
| `region1`  | `zone2` | 20              |
| `region2`  | `zone3` | 0               |
| `region3`  | `zone4` | 10              |

## 重み付き分散の構成 {#configure-weighted-distribution}

`DestinationRule` を次のように適用します：

- `HelloWorld` サービスに対して[異常検出](/ja/docs/reference/config/networking/destination-rule/#OutlierDetection)を有効にします。
  これは分散が正しく動作するために必須です。
  特に、Sidecar プロキシがサービスのエンドポイントのヘルス状態を把握できるようにします。

- [重み付き分散](https://www.envoyproxy.io/docs/envoy/latest/intro/arch_overview/upstream/load_balancing/locality_weight.html?highlight=weight)
  を上記表の通り `HelloWorld` サービスに設定します。

{{< text bash >}}
$ kubectl --context="${CTX_PRIMARY}" apply -n sample -f - <<EOF
apiVersion: networking.istio.io/v1
kind: DestinationRule
metadata:
name: helloworld
spec:
host: helloworld.sample.svc.cluster.local
trafficPolicy:
loadBalancer:
localityLbSetting:
enabled: true
distribute: - from: region1/zone1/_
to:
"region1/zone1/_": 70
"region1/zone2/_": 20
"region3/zone4/_": 10
outlierDetection:
consecutive5xxErrors: 100
interval: 1s
baseEjectionTime: 1m
EOF
{{< /text >}}

## 分散の検証 {#verify-the-distribution}

`curl` Pod から `HelloWorld` サービスを呼び出します：

{{< text bash >}}
$ kubectl exec --context="${CTX_R1_Z1}" -n sample -c curl \
  "$(kubectl get pod --context="${CTX_R1_Z1}" -n sample -l \
 app=curl -o jsonpath='{.items[0].metadata.name}')" \
 -- curl -sSL helloworld.sample:5000/hello
{{< /text >}}

これを何度も繰り返し、各 Pod の応答数が冒頭の表に記載された期待値に近いことを確認してください。

**おめでとうございます！** ローカリティ重み付き分散の構成に成功しました！

## 次のステップ {#next-steps}

[クリーンアップ](/ja/docs/tasks/traffic-management/locality-load-balancing/cleanup) タスクでファイルやリソースを削除してください。
