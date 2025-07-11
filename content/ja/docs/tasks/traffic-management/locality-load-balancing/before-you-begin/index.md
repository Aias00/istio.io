---
title: 始める前に
description: ローカリティ負荷分散を構成する前の初期化手順。
weight: 1
icon: tasks
keywords:
  [locality, load balancing, priority, prioritized, kubernetes, multicluster]
test: yes
owner: istio/wg-networking-maintainers
---

ローカリティ負荷分散タスクを始める前に、まず[複数クラスタに Istio をインストール](/ja/docs/setup/install/multicluster)する必要があります。
これらのクラスタは 3 つのリージョンにまたがり、4 つのアベイラビリティゾーンを含みます。
必要なクラスタ数は、クラウドプロバイダーの機能によって異なる場合があります。

{{< tip >}}
簡単のため、メッシュ内に {{< gloss >}}プライマリクラスタ{{< /gloss >}} が 1 つだけあると仮定します。
変更は 1 つのクラスタにのみ適用すればよいので、コントロールプレーンの構成が簡単になります。
{{< /tip >}}

`HelloWorld` アプリケーションの複数インスタンスを次のようにデプロイします：

{{< image width="75%"
    link="setup.svg"
    caption="ローカリティ負荷分散タスクのセットアップ"
    >}}

{{< tip >}}
単一のマルチリージョンクラスタ環境でも、同じクラスタ内の異なるリージョンへのフェイルオーバーのためにローカリティ負荷分散を構成できます。
これをテストするには、複数のワークゾーンを持つクラスタを作成し、各ゾーンに istiod インスタンスとアプリケーションをデプロイする必要があります。

1: 複数リージョンの Kubernetes クラスタがない場合は、`kind` を使って以下のコマンドでローカルにクラスタをデプロイできます：

{{< text syntax=bash snip_id=none >}}
$ kind create cluster --config=- <<EOF
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:

- role: control-plane
- role: worker
- role: worker
- role: worker
  EOF
  {{< /text >}}

2: 各ワーカーノードに `topology.kubernetes.io/zone` でゾーン名のラベルを追加します：

{{< text syntax=bash snip_id=none >}}
$ kubectl label node kind-worker topology.kubernetes.io/zone=us-south10
$ kubectl label node kind-worker2 topology.kubernetes.io/zone=us-south12
$ kubectl label node kind-worker3 topology.kubernetes.io/zone=us-south13
{{< /text >}}

3: istiod をコントロールプレーンノードにデプロイし、各ワーカーノードに helloworld アプリケーションをデプロイします。

{{< /tip >}}

## 環境変数 {#environment-variables}

本ガイドでは、すべてのクラスタへのアクセスにデフォルトの [Kubernetes 設定ファイル](https://kubernetes.io/ja/docs/tasks/access-application-cluster/configure-access-multiple-clusters/) のコンテキストを使用することを前提としています。
以下の環境変数は各種コンテキストで使用します：

| 変数          | 説明                                                      |
| ------------- | --------------------------------------------------------- |
| `CTX_PRIMARY` | プライマリクラスタ用のコンテキスト。                      |
| `CTX_R1_Z1`   | `region1.zone1` の Pod とやり取りするためのコンテキスト。 |
| `CTX_R1_Z2`   | `region1.zone2` の Pod とやり取りするためのコンテキスト。 |
| `CTX_R2_Z3`   | `region2.zone3` の Pod とやり取りするためのコンテキスト。 |
| `CTX_R3_Z4`   | `region3.zone4` の Pod とやり取りするためのコンテキスト。 |

## `sample` 名前空間の作成 {#create-the-sample-namespace}

まず、Sidecar の自動インジェクションを有効にし、`sample` 名前空間の yaml を生成します：

{{< text bash >}}
$ cat <<EOF > sample.yaml
apiVersion: v1
kind: Namespace
metadata:
name: sample
labels:
istio-injection: enabled
EOF
{{< /text >}}

各クラスタに `sample` 名前空間を追加します：

{{< text bash >}}
$ for CTX in "$CTX_PRIMARY" "$CTX_R1_Z1" "$CTX_R1_Z2" "$CTX_R2_Z3" "$CTX_R3_Z4"; \
  do \
    kubectl --context="$CTX" apply -f sample.yaml; \
 done
{{< /text >}}

## `HelloWorld` のデプロイ {#deploy-helloWorld}

リージョン名をバージョン番号として、各リージョン用の `HelloWorld` の yaml を生成します：

{{< text bash >}}
$ for LOC in "region1.zone1" "region1.zone2" "region2.zone3" "region3.zone4"; \
 do \
 ./@samples/helloworld/gen-helloworld.sh@ \
 --version "$LOC" > "helloworld-${LOC}.yaml"; \
 done
{{< /text >}}

各リージョンの適切なクラスタに `HelloWorld` YAML を適用します：

{{< text bash >}}
$ kubectl apply --context="${CTX_R1_Z1}" -n sample \
 -f helloworld-region1.zone1.yaml
{{< /text >}}

{{< text bash >}}
$ kubectl apply --context="${CTX_R1_Z2}" -n sample \
 -f helloworld-region1.zone2.yaml
{{< /text >}}

{{< text bash >}}
$ kubectl apply --context="${CTX_R2_Z3}" -n sample \
 -f helloworld-region2.zone3.yaml
{{< /text >}}

{{< text bash >}}
$ kubectl apply --context="${CTX_R3_Z4}" -n sample \
 -f helloworld-region3.zone4.yaml
{{< /text >}}

## `curl` のデプロイ {#deploy-curl}

`curl` アプリケーションを `region1` `zone1` にデプロイします：

{{< text bash >}}
$ kubectl apply --context="${CTX_R1_Z1}" \
 -f @samples/curl/curl.yaml@ -n sample
{{< /text >}}

## `HelloWorld` Pod の待機 {#wait-for-helloworld-pods}

各リージョンの Pod で `HelloWorld` が `Running` になるまで待ちます：

{{< text bash >}}
$ kubectl get pod --context="${CTX_R1_Z1}" -n sample -l app="helloworld" \
 -l version="region1.zone1"
NAME READY STATUS RESTARTS AGE
helloworld-region1.zone1-86f77cd7b-cpxhv 2/2 Running 0 30s
{{< /text >}}

{{< text bash >}}
$ kubectl get pod --context="${CTX_R1_Z2}" -n sample -l app="helloworld" \
 -l version="region1.zone2"
NAME READY STATUS RESTARTS AGE
helloworld-region1.zone2-86f77cd7b-cpxhv 2/2 Running 0 30s
{{< /text >}}

{{< text bash >}}
$ kubectl get pod --context="${CTX_R2_Z3}" -n sample -l app="helloworld" \
 -l version="region2.zone3"
NAME READY STATUS RESTARTS AGE
helloworld-region2.zone3-86f77cd7b-cpxhv 2/2 Running 0 30s
{{< /text >}}

{{< text bash >}}
$ kubectl get pod --context="${CTX_R3_Z4}" -n sample -l app="helloworld" \
 -l version="region3.zone4"
NAME READY STATUS RESTARTS AGE
helloworld-region3.zone4-86f77cd7b-cpxhv 2/2 Running 0 30s
{{< /text >}}

**おめでとうございます！** システム構成が完了し、ローカリティ負荷分散タスクを始める準備ができました！

## 次のステップ {#next-steps}

これで、以下のいずれかの負荷分散オプションを構成できます：

- [ローカリティフェイルオーバー](/ja/docs/tasks/traffic-management/locality-load-balancing/failover)

- [ローカリティ重み付き分散](/ja/docs/tasks/traffic-management/locality-load-balancing/distribute)

{{< warning >}}
これらのオプションは排他的なので、どちらか一方のみを構成してください。両方を同時に構成しようとすると予期しない動作になる場合があります。
{{< /warning >}}
