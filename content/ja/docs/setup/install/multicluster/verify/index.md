---
title: インストール結果の検証
description: Istio がマルチクラスタ環境に正常にインストールされたことを検証します。
weight: 50
icon: setup
keywords: [kubernetes, multicluster]
test: no
owner: istio/wg-environments-maintainers
---

このガイドに従って、マルチクラスタ環境にインストールされた Istio が正常に動作するかを検証します。

操作を続ける前に、[事前準備](/zh/docs/setup/install/multicluster/before-you-begin)の手順を完了していることを確認してください。

このガイドでは、マルチクラスタが正常に動作しているかを検証します。
`HelloWorld` アプリケーションの `V1` を `cluster1` に、
`V2` を `cluster2` にデプロイします。リクエストを受け取ると、`HelloWorld` はレスポンスに自身のバージョンを含めます。

また、両方のクラスタに `curl` コンテナもデプロイします。
これらの Pod はクライアント（source）として機能し、`HelloWorld` へリクエストを送信します。
最後に、これらのトラフィックデータを収集することで、どのクラスタがリクエストを処理したかを観測・識別できます。

## マルチクラスタの検証 {#verify-multicluster}

Istiod がリモートクラスタの Kubernetes コントロールプレーンと通信できることを確認します。

{{< text bash >}}
$ istioctl remote-clusters --context="${CTX_CLUSTER1}"
NAME SECRET STATUS ISTIOD
cluster1 synced istiod-7b74b769db-kb4kj
cluster2 istio-system/istio-remote-secret-cluster2 synced istiod-7b74b769db-kb4kj
{{< /text >}}

すべてのクラスタのステータスが `synced` である必要があります。クラスタの `STATUS` が `timeout` の場合、
プライマリクラスタの Istiod がリモートクラスタと通信できていません。詳細なエラーメッセージは Istiod のログを参照してください。

注意：もし `timeout` の問題が発生し、
プライマリクラスタの Istiod とリモートクラスタの Kubernetes
コントロールプレーンの間に中間ホスト（例: [Rancher 認証プロキシ](https://ranchermanager.docs.rancher.com/how-to-guides/new-user-guides/manage-clusters/access-clusters/authorized-cluster-endpoint#two-authentication-methods-for-rke-clusters)）が存在する場合、
`istioctl create-remote-secret` で生成された kubeconfig の
`certificate-authority-data` フィールドを中間ホストで使用されている証明書に合わせて更新する必要があります。

## サービス `HelloWorld` のデプロイ {#deploy-the-helloworld-service}

任意のクラスタから `HelloWorld` サービスを呼び出せるようにするため、各クラスタの DNS 解決が利用可能である必要があります
（詳細は[デプロイメントモデル](/zh/docs/ops/deployment/deployment-models#dns-with-multiple-clusters)を参照）。
この問題を解決するため、メッシュ内のすべてのクラスタに `HelloWorld` サービスをデプロイします。

まず、各クラスタに `sample` ネームスペースを作成します：

{{< text bash >}}
$ kubectl create --context="${CTX_CLUSTER1}" namespace sample
$ kubectl create --context="${CTX_CLUSTER2}" namespace sample
{{< /text >}}

`sample` ネームスペースで sidecar の自動インジェクションを有効にします：

{{< text bash >}}
$ kubectl label --context="${CTX_CLUSTER1}" namespace sample \
    istio-injection=enabled
$ kubectl label --context="${CTX_CLUSTER2}" namespace sample \
 istio-injection=enabled
{{< /text >}}

各クラスタで `HelloWorld` サービスを作成します：

{{< text bash >}}
$ kubectl apply --context="${CTX_CLUSTER1}" \
    -f @samples/helloworld/helloworld.yaml@ \
    -l service=helloworld -n sample
$ kubectl apply --context="${CTX_CLUSTER2}" \
 -f @samples/helloworld/helloworld.yaml@ \
 -l service=helloworld -n sample
{{< /text >}}

## `HelloWorld` の `V1` バージョンをデプロイ {#deploy-helloworld-v1}

`helloworld-v1` アプリケーションを `cluster1` にデプロイします：

{{< text bash >}}
$ kubectl apply --context="${CTX_CLUSTER1}" \
 -f @samples/helloworld/helloworld.yaml@ \
 -l version=v1 -n sample
{{< /text >}}

`helloworld-v1` pod の状態を確認します：

{{< text bash >}}
$ kubectl get pod --context="${CTX_CLUSTER1}" -n sample -l app=helloworld
NAME READY STATUS RESTARTS AGE
helloworld-v1-86f77cd7bd-cpxhv 2/2 Running 0 40s
{{< /text >}}

`helloworld-v1` の状態が最終的に `Running` になるまで待ちます：

## `HelloWorld` の `V2` バージョンをデプロイ {#deploy-helloworld-v1}

`helloworld-v2` アプリケーションを `cluster2` にデプロイします：

{{< text bash >}}
$ kubectl apply --context="${CTX_CLUSTER2}" \
 -f @samples/helloworld/helloworld.yaml@ \
 -l version=v2 -n sample
{{< /text >}}

`helloworld-v2` pod の状態を確認します：

{{< text bash >}}
$ kubectl get pod --context="${CTX_CLUSTER2}" -n sample -l app=helloworld
NAME READY STATUS RESTARTS AGE
helloworld-v2-758dd55874-6x4t8 2/2 Running 0 40s
{{< /text >}}

`helloworld-v2` の状態が最終的に `Running` になるまで待ちます：

## `curl` のデプロイ {#deploy-curl}

各クラスタに `curl` アプリケーションをデプロイします：

{{< text bash >}}
$ kubectl apply --context="${CTX_CLUSTER1}" \
    -f @samples/curl/curl.yaml@ -n sample
$ kubectl apply --context="${CTX_CLUSTER2}" \
 -f @samples/curl/curl.yaml@ -n sample
{{< /text >}}

`cluster1` 上の `curl` の状態を確認します：

{{< text bash >}}
$ kubectl get pod --context="${CTX_CLUSTER1}" -n sample -l app=curl
NAME READY STATUS RESTARTS AGE
curl-754684654f-n6bzf 2/2 Running 0 5s
{{< /text >}}

`curl` の状態が最終的に `Running` になるまで待ちます：

`cluster2` 上の `curl` の状態を確認します：

{{< text bash >}}
$ kubectl get pod --context="${CTX_CLUSTER2}" -n sample -l app=curl
NAME READY STATUS RESTARTS AGE
curl-754684654f-dzl9j 2/2 Running 0 5s
{{< /text >}}

`curl` の状態が最終的に `Running` になるまで待ちます：

## クラスタ間トラフィックの検証 {#verifying-cross-cluster-traffic}

クラスタ間の負荷分散が期待通りに動作しているかを検証するには、`curl` pod から `HelloWorld` サービスを繰り返し呼び出します。
負荷分散が期待通りに動作していることを確認するため、すべてのクラスタから `HelloWorld` サービスを呼び出します。

`cluster1` の `curl` pod から `HelloWorld` サービスにリクエストを送信します：

{{< text bash >}}
$ kubectl exec --context="${CTX_CLUSTER1}" -n sample -c curl \
    "$(kubectl get pod --context="${CTX_CLUSTER1}" -n sample -l \
 app=curl -o jsonpath='{.items[0].metadata.name}')" \
 -- curl helloworld.sample:5000/hello
{{< /text >}}

このリクエストを数回繰り返し、`HelloWorld` のバージョンが `v1` と `v2` の間で切り替わることを確認します：

{{< text plain >}}
Hello version: v2, instance: helloworld-v2-758dd55874-6x4t8
Hello version: v1, instance: helloworld-v1-86f77cd7bd-cpxhv
...
{{< /text >}}

次に、`cluster2` の `curl` pod でも同じ操作を繰り返します：

{{< text bash >}}
$ kubectl exec --context="${CTX_CLUSTER2}" -n sample -c curl \
    "$(kubectl get pod --context="${CTX_CLUSTER2}" -n sample -l \
 app=curl -o jsonpath='{.items[0].metadata.name}')" \
 -- curl helloworld.sample:5000/hello
{{< /text >}}

このリクエストを数回繰り返し、`HelloWorld` のバージョンが `v1` と `v2` の間で切り替わることを確認します：

{{< text plain >}}
Hello version: v2, instance: helloworld-v2-758dd55874-6x4t8
Hello version: v1, instance: helloworld-v1-86f77cd7bd-cpxhv
...
{{< /text >}}

**おめでとうございます！** マルチクラスタ環境に Istio をインストールし、検証に成功しました！

## 次のステップ {#next-steps}

[ローカリティベースの負荷分散タスク](/zh/docs/tasks/traffic-management/locality-load-balancing)を参照し、
マルチクラスタメッシュ間でのトラフィック制御方法を学びましょう。
