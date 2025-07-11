---
title: ドライラン（シミュレーション実行）
description: 実際に適用せずに認可ポリシーの効果を観察する方法を紹介します。
weight: 65
keywords: [security, access-control, rbac, authorization, dry-run]
owner: istio/wg-security-maintainers
test: yes
status: Alpha
---

{{< boilerplate alpha >}}

このタスクでは、新しい[実験的アノテーション `istio.io/dry-run`](/ja/docs/reference/config/annotations/)を使って、Istio 認可ポリシーをシミュレーション実行（ドライラン）し、実際に適用せずにその効果を確認する方法を紹介します。

ドライランアノテーションを使うことで、本番トラフィックに認可ポリシーを適用する前にその影響をよりよく理解でき、
誤った認可ポリシーによる本番トラフィックの中断リスクを減らすのに役立ちます。

## 始める前に {#before-you-begin}

このタスクを始める前に、以下を完了してください：

- [Istio 認可の概念](/ja/docs/concepts/security/#authorization)を読んでください。

- [Istio インストールガイド](/ja/docs/setup/install)に従って Istio をインストールしてください。

- Zipkin をデプロイしてドライランのトレース結果を確認します。
  [Zipkin タスク](/ja/docs/tasks/observability/distributed-tracing/zipkin/)に従って Zipkin をクラスタにインストールしてください。

- Prometheus をデプロイしてドライランのメトリクス結果を確認します。
  [Prometheus タスク](/ja/docs/tasks/observability/metrics/querying-metrics/)に従って Prometheus をクラスタにインストールしてください。

- テスト用ワークロードをデプロイします：

  このタスクでは、`foo` 名前空間に `httpbin` と `curl` の 2 つのワークロードを使用します。
  どちらも Envoy サイドカーを持ちます。以下のコマンドで `foo` 名前空間を作成し、ワークロードをデプロイします：

  {{< text bash >}}
  $ kubectl create ns foo
  $ kubectl label ns foo istio-injection=enabled
  $ kubectl apply -f @samples/httpbin/httpbin.yaml@ -n foo
  $ kubectl apply -f @samples/curl/curl.yaml@ -n foo
  {{< /text >}}

- ドライランのログ結果を確認するため、プロキシのデバッグレベルログを有効にします：

  {{< text bash >}}
  $ istioctl proxy-config log deploy/httpbin.foo --level "rbac:debug" | grep rbac
  rbac: debug
  {{< /text >}}

- 次のコマンドで `curl` から `httpbin` へのアクセスを確認します：

  {{< text bash >}}
  $ kubectl exec "$(kubectl get pod -l app=curl -n foo -o jsonpath={.items..metadata.name})" -c curl -n foo -- curl http://httpbin.foo:8000/ip -s -o /dev/null -w "%{http_code}\n"
  200
  {{< /text >}}

{{< warning >}}
ガイド通りに操作しても期待される出力が得られない場合は、数秒待ってから再試行してください。
キャッシュや伝播のオーバーヘッドにより遅延が発生することがあります。
{{< /warning >}}

## ドライランポリシーの作成 {#create-dry-run-policy}

1. 次のコマンドで、ドライランアノテーション `"istio.io/dry-run": "true"` を持つ認可ポリシーを作成します：

   {{< text bash >}}
   $ kubectl apply -n foo -f - <<EOF
   apiVersion: security.istio.io/v1
   kind: AuthorizationPolicy
   metadata:
   name: deny-path-headers
   annotations:
   "istio.io/dry-run": "true"
   spec:
   selector:
   matchLabels:
   app: httpbin
   action: DENY
   rules:

   - to: - operation:
     paths: ["/headers"]
     EOF
     {{< /text >}}

   既存の認可ポリシーをドライランモードに素早く変更するには、次のコマンドを使います：

   {{< text bash >}}
   $ kubectl annotate --overwrite authorizationpolicies deny-path-headers -n foo istio.io/dry-run='true'
   {{< /text >}}

1. ポリシーがドライランモードで作成されているため、`/headers` パスへのリクエストが許可されることを確認します。
   次のコマンドで `curl` から `httpbin` へ 20 回リクエストを送信します。
   リクエストには `X-B3-Sampled: 1` ヘッダーを付与し、常に Zipkin トレースを発生させます：

   {{< text bash >}}
   $ for i in {1..20}; do kubectl exec "$(kubectl get pod -l app=curl -n foo -o jsonpath={.items..metadata.name})" -c curl -n foo -- curl http://httpbin.foo:8000/headers -H "X-B3-Sampled: 1" -s -o /dev/null -w "%{http_code}\n"; done
   200
   200
   200
   ...
   {{< /text >}}

## プロキシログでドライラン結果を確認する {#check-dry-run-results-in-proxy-log}

ドライラン結果はプロキシのデバッグログで確認できます。ログの形式は
`shadow denied, matched policy ns[foo]-policy[deny-path-headers]-rule[0]` です。
次のコマンドでログを確認します：

{{< text bash >}}
$ kubectl logs "$(kubectl -n foo -l app=httpbin get pods -o jsonpath={.items..metadata.name})" -c istio-proxy -n foo | grep "shadow denied"
2021-11-19T20:20:48.733099Z debug envoy rbac shadow denied, matched policy ns[foo]-policy[deny-path-headers]-rule[0]
2021-11-19T20:21:45.502199Z debug envoy rbac shadow denied, matched policy ns[foo]-policy[deny-path-headers]-rule[0]
2021-11-19T20:22:33.065348Z debug envoy rbac shadow denied, matched policy ns[foo]-policy[deny-path-headers]-rule[0]
...
{{< /text >}}

詳細は[トラブルシューティングガイド](/ja/docs/ops/common-problems/security-issues/#ensure-proxies-enforce-policies-correctly)も参照してください。

## Prometheus でメトリクスのドライラン結果を確認する {#check-dry-run-result-in-metric-using-prometheus}

1. 次のコマンドで Prometheus ダッシュボードを開きます：

   {{< text bash >}}
   $ istioctl dashboard prometheus
   {{< /text >}}

1. Prometheus ダッシュボードで次のメトリクスを検索します：

   {{< text plain >}}
   envoy_http_inbound_0_0_0_0_80_rbac{authz_dry_run_action="deny",authz_dry_run_result="denied"}
   {{< /text >}}

1. 次のようなクエリ結果を確認します：

   {{< text plain >}}
   envoy_http_inbound_0_0_0_0_80_rbac{app="httpbin",authz_dry_run_action="deny",authz_dry_run_result="denied",instance="10.44.1.11:15020",istio_io_rev="default",job="kubernetes-pods",kubernetes_namespace="foo",kubernetes_pod_name="httpbin-74fb669cc6-95qm8",pod_template_hash="74fb669cc6",security_istio_io_tlsMode="istio",service_istio_io_canonical_name="httpbin",service_istio_io_canonical_revision="v1",version="v1"} 20
   {{< /text >}}

1. クエリの値が `20` であることを確認します（リクエスト数によって異なる場合がありますが、0 より大きければ期待通りです）。
   これは、ドライランポリシーがポート `80` 上の `httpbin` ワークロードでリクエストにマッチしたことを意味します。
   ポリシーがドライランでなければ、リクエストは 1 回拒否されます。

1. Prometheus ダッシュボードのスクリーンショット例：

   {{< image width="100%" link="./prometheus.png" caption="Prometheus dashboard" >}}

## Zipkin でトレースのドライラン結果を確認する {#check-dry-run-result-in-tracing-using-zipkin}

1. 次のコマンドで Zipkin ダッシュボードを開きます：

   {{< text bash >}}
   $ istioctl dashboard zipkin
   {{< /text >}}

1. `curl` から `httpbin` へのリクエストのトレース結果を探します。
   Zipkin の遅延で結果が見つからない場合は、リクエストを追加で送信してください。

1. トレース結果に、次のカスタムタグが表示されていることを確認します。これは、ネームスペース `foo` のドライランポリシー `deny-path-headers` でリクエストが拒否されたことを示します：

   {{< text plain >}}
   istio.authorization.dry_run.deny_policy.name: ns[foo]-policy[deny-path-headers]-rule[0]
   istio.authorization.dry_run.deny_policy.result: denied
   {{< /text >}}

1. Zipkin ダッシュボードのスクリーンショット例：

   {{< image width="100%" link="./trace.png" caption="Zipkin dashboard" >}}

## まとめ {#summary}

プロキシのデバッグログ、Prometheus メトリクス、Zipkin トレース結果から、ドライランポリシーがリクエストを拒否することが分かります。
ドライラン結果が期待と異なる場合は、ポリシーをさらに調整してください。

本番トラフィックで十分にテストできるよう、しばらくドライランポリシーを維持することを推奨します。

ドライラン結果に自信が持てたら、ドライランモードを無効にしてポリシーを実際に適用できます。次のいずれかの方法で実現できます：

- ドライランアノテーションを完全に削除する；または

- ドライランアノテーションの値を `false` に変更する。

## 制限事項 {#limiatations}

ドライランアノテーションは現在実験的で、以下の制限があります：

- ドライランアノテーションは現在 ALLOW および DENY ポリシーのみサポートしています；

- プロキシ内で ALLOW と DENY ポリシーが独立して実行されるため、2 つのドライラン結果（ログ・メトリクス・トレースタグ）が出力されます。
  両方のドライラン結果を考慮してください。1 つのリクエストが ALLOW ポリシーで許可されても、DENY ポリシーで拒否される場合があります；

- プロキシログ・メトリクス・トレースのドライラン結果は手動トラブルシューティング用であり、API として利用すべきではありません（予告なく変更される場合があります）。

## クリーンアップ {#clean-up}

1. 設定から `foo` 名前空間を削除します：

   {{< text bash >}}
   $ kubectl delete namespace foo
   {{< /text >}}

1. 不要であれば Prometheus と Zipkin も削除してください。
