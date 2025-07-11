---
title: 単一クラスタで複数の Istio コントロールプレーンをインストールする
description: リビジョンと discoverySelectors を使って単一クラスタに複数の Istio コントロールプレーンをインストールします。
weight: 55
keywords: [multiple, control, istiod, local]
owner: istio/wg-environments-maintainers
test: yes
---

{{< boilerplate experimental-feature-warning >}}

このガイドでは、単一クラスタに複数の Istio コントロールプレーンをインストールする手順と、ワークロードのスコープを特定のコントロールプレーンに割り当てる方法を説明します。
このデプロイメントモデルは、単一の Kubernetes コントロールプレーンと複数の Istio コントロールプレーンおよび複数のメッシュを利用します。
メッシュ間の分離は、Kubernetes のネームスペースと RBAC によって実現されます。

{{< image width="90%"
    link="single-cluster-multiple-istiods.svg"
    caption="単一クラスタ内の複数メッシュ"
    >}}

`discoverySelectors` を使うことで、クラスタ内の Kubernetes リソースのスコープを特定の Istio コントロールプレーンが管理するネームスペースに限定できます。
これには、メッシュの設定に使われる Istio カスタムリソース（Gateway、VirtualService、DestinationRule など）が含まれます。
さらに、`discoverySelectors` を使って、どのネームスペースに特定の Istio コントロールプレーン用の `istio-ca-root-cert` ConfigMap を含めるかを設定できます。
これらの機能により、メッシュ管理者はコントロールプレーンごとにネームスペースを指定でき、1 つまたは複数のネームスペースの境界に基づいて複数メッシュのソフトなマルチテナンシーを実現できます。
このガイドでは、`discoverySelectors` と Istio のリビジョン機能を使い、単一クラスタ上に 2 つのメッシュをデプロイし、それぞれが適切なスコープのクラスタリソースサブセットを利用する方法を説明します。

## 始める前に{#before-you-begin}

このガイドでは、[サポートされている Kubernetes バージョン](/zh/docs/releases/supported-releases#support-status-of-istio-releases) {{< supported_kubernetes_versions >}} のいずれかがインストールされた Kubernetes クラスタが必要です。

このクラスタには、2 つの異なるシステムネームスペースにインストールされた 2 つのコントロールプレーンが含まれます。
メッシュアプリケーションのワークロードは、複数のアプリケーション専用ネームスペースで実行され、各ネームスペースはリビジョンとディスカバリセレクターの設定に基づいていずれかのコントロールプレーンと関連付けられます。

## クラスタ構成{#cluster-configuration}

### 複数コントロールプレーンのデプロイ{#deploying-multiple-control-planes}

単一クラスタ上に複数の Istio コントロールプレーンをデプロイするには、それぞれ異なるシステムネームスペースを使用します。
Istio のリビジョンと `discoverySelectors` を使って、各コントロールプレーンが管理するリソースやワークロードのスコープを決定します。

1. 最初のシステムネームスペース `usergroup-1` を作成し、そこに istiod をデプロイします：

   {{< text bash >}}
   $ kubectl create ns usergroup-1
   $ kubectl label ns usergroup-1 usergroup=usergroup-1
   $ istioctl install -y -f - <<EOF
   apiVersion: install.istio.io/v1alpha1
   kind: IstioOperator
   metadata:
   namespace: usergroup-1
   spec:
   profile: minimal
   revision: usergroup-1
   meshConfig:
   discoverySelectors: - matchLabels:
   usergroup: usergroup-1
   values:
   global:
   istioNamespace: usergroup-1
   EOF
   {{< /text >}}

1. 2 つ目のシステムネームスペース `usergroup-2` を作成し、そこに istiod をデプロイします：

   {{< text bash >}}
   $ kubectl create ns usergroup-2
   $ kubectl label ns usergroup-2 usergroup=usergroup-2
   $ istioctl install -y -f - <<EOF
   apiVersion: install.istio.io/v1alpha1
   kind: IstioOperator
   metadata:
   namespace: usergroup-2
   spec:
   profile: minimal
   revision: usergroup-2
   meshConfig:
   discoverySelectors: - matchLabels:
   usergroup: usergroup-2
   values:
   global:
   istioNamespace: usergroup-2
   EOF
   {{< /text >}}

1. `usergroup-1` ネームスペースのワークロードに対して、双方向 TLS トラフィックのみを受け入れるようにポリシーをデプロイします：

   {{< text bash >}}
   $ kubectl apply -f - <<EOF
   apiVersion: security.istio.io/v1
   kind: PeerAuthentication
   metadata:
   name: "usergroup-1-peerauth"
   namespace: "usergroup-1"
   spec:
   mtls:
   mode: STRICT
   EOF
   {{< /text >}}

1. `usergroup-2` ネームスペースのワークロードに対して、双方向 TLS トラフィックのみを受け入れるようにポリシーをデプロイします：

   {{< text bash >}}
   $ kubectl apply -f - <<EOF
   apiVersion: security.istio.io/v1
   kind: PeerAuthentication
   metadata:
   name: "usergroup-2-peerauth"
   namespace: "usergroup-2"
   spec:
   mtls:
   mode: STRICT
   EOF
   {{< /text >}}

### 複数コントロールプレーン作成の確認{#verify-multiple-control-plane-creation}

1. 各コントロールプレーンのシステムネームスペースのラベルを確認します：

   {{< text bash >}}
   $ kubectl get ns usergroup-1 usergroup-2 --show-labels
   NAME STATUS AGE LABELS
   usergroup-1 Active 13m kubernetes.io/metadata.name=usergroup-1,usergroup=usergroup-1
   usergroup-2 Active 12m kubernetes.io/metadata.name=usergroup-2,usergroup=usergroup-2
   {{< /text >}}

1. コントロールプレーンがデプロイされ、稼働していることを確認します：

   {{< text bash >}}
   $ kubectl get pods -n usergroup-1
   NAMESPACE NAME READY STATUS RESTARTS AGE
   usergroup-1 istiod-usergroup-1-5ccc849b5f-wnqd6 1/1 Running 0 12m
   {{< /text >}}

   {{< text bash >}}
   $ kubectl get pods -n usergroup-2
   NAMESPACE NAME READY STATUS RESTARTS AGE
   usergroup-2 istiod-usergroup-2-658d6458f7-slpd9 1/1 Running 0 12m
   {{< /text >}}

   指定したネームスペースごとに、各ユーザーグループ用の Istiod Deployment が作成されていることが分かります。

1. 次のコマンドでインストール済みの Webhook を一覧表示します：

   {{< text bash >}}
   $ kubectl get validatingwebhookconfiguration
   NAME WEBHOOKS AGE
   istio-validator-usergroup-1-usergroup-1 1 18m
   istio-validator-usergroup-2-usergroup-2 1 18m
   istiod-default-validator 1 18m
   {{< /text >}}

   {{< text bash >}}
   $ kubectl get mutatingwebhookconfiguration
   NAME WEBHOOKS AGE
   istio-revision-tag-default-usergroup-1 4 18m
   istio-sidecar-injector-usergroup-1-usergroup-1 2 19m
   istio-sidecar-injector-usergroup-2-usergroup-2 2 18m
   {{< /text >}}

   出力には `istiod-default-validator` や `istio-revision-tag-default-usergroup-1` など、リビジョンに関連付けられていないリソースリクエストを処理するためのデフォルト Webhook 設定が含まれています。
   フルスコープ環境では、各コントロールプレーンは適切なネームスペースラベルによってリソースと関連付けられているため、これらのデフォルト Webhook 設定は不要です。
   それらは呼び出されるべきではありません。

### 各ユーザーグループごとにアプリケーションワークロードをデプロイ{#deploy-app-workloads-per-usergroup}

1. 3 つのアプリケーションネームスペースを作成します：

   {{< text bash >}}
   $ kubectl create ns app-ns-1
   $ kubectl create ns app-ns-2
   $ kubectl create ns app-ns-3
   {{< /text >}}

1. 各ネームスペースにラベルを付与し、それぞれのコントロールプレーンと関連付けます：

   {{< text bash >}}
   $ kubectl label ns app-ns-1 usergroup=usergroup-1 istio.io/rev=usergroup-1
   $ kubectl label ns app-ns-2 usergroup=usergroup-2 istio.io/rev=usergroup-2
   $ kubectl label ns app-ns-3 usergroup=usergroup-2 istio.io/rev=usergroup-2
   {{< /text >}}

1. 各ネームスペースに `curl` と `httpbin` アプリケーションをデプロイします：

   {{< text bash >}}
   $ kubectl -n app-ns-1 apply -f samples/curl/curl.yaml
   $ kubectl -n app-ns-1 apply -f samples/httpbin/httpbin.yaml
   $ kubectl -n app-ns-2 apply -f samples/curl/curl.yaml
   $ kubectl -n app-ns-2 apply -f samples/httpbin/httpbin.yaml
   $ kubectl -n app-ns-3 apply -f samples/curl/curl.yaml
   $ kubectl -n app-ns-3 apply -f samples/httpbin/httpbin.yaml
   {{< /text >}}

1. 数秒待ち、`httpbin` と `curl` の Pod が Sidecar を注入された状態で稼働していることを確認します：

   {{< text bash >}}
   $ kubectl get pods -n app-ns-1
   NAME READY STATUS RESTARTS AGE
   httpbin-9dbd644c7-zc2v4 2/2 Running 0 115m
   curl-78ff5975c6-fml7c 2/2 Running 0 115m
   {{< /text >}}

   {{< text bash >}}
   $ kubectl get pods -n app-ns-2
   NAME READY STATUS RESTARTS AGE
   httpbin-9dbd644c7-sd9ln 2/2 Running 0 115m
   curl-78ff5975c6-sz728 2/2 Running 0 115m
   {{< /text >}}

   {{< text bash >}}
   $ kubectl get pods -n app-ns-3
   NAME READY STATUS RESTARTS AGE
   httpbin-9dbd644c7-8ll27 2/2 Running 0 115m
   curl-78ff5975c6-sg4tq 2/2 Running 0 115m
   {{< /text >}}

### アプリケーションとコントロールプレーンのマッピング確認{#verify-app-to-control-plane-mapping}

アプリケーションがデプロイされたら、`istioctl ps` コマンドを使って、各アプリケーションワークロードがそれぞれのコントロールプレーンによって管理されていることを確認できます。
つまり、`app-ns-1` は `usergroup-1` に、`app-ns-2` と `app-ns-3` は `usergroup-2` によって管理されています：

{{< text bash >}}
$ istioctl ps -i usergroup-1
NAME CLUSTER CDS LDS EDS RDS ECDS ISTIOD VERSION
httpbin-9dbd644c7-hccpf.app-ns-1 Kubernetes SYNCED SYNCED SYNCED SYNCED NOT SENT istiod-usergroup-1-5ccc849b5f-wnqd6 1.17-alpha.f5212a6f7df61fd8156f3585154bed2f003c4117
curl-78ff5975c6-9zb77.app-ns-1 Kubernetes SYNCED SYNCED SYNCED SYNCED NOT SENT istiod-usergroup-1-5ccc849b5f-wnqd6 1.17-alpha.f5212a6f7df61fd8156f3585154bed2f003c4117
{{< /text >}}

{{< text bash >}}
$ istioctl ps -i usergroup-2
NAME CLUSTER CDS LDS EDS RDS ECDS ISTIOD VERSION
httpbin-9dbd644c7-vvcqj.app-ns-3 Kubernetes SYNCED SYNCED SYNCED SYNCED NOT SENT istiod-usergroup-2-658d6458f7-slpd9 1.17-alpha.f5212a6f7df61fd8156f3585154bed2f003c4117
httpbin-9dbd644c7-xzgfm.app-ns-2 Kubernetes SYNCED SYNCED SYNCED SYNCED NOT SENT istiod-usergroup-2-658d6458f7-slpd9 1.17-alpha.f5212a6f7df61fd8156f3585154bed2f003c4117
curl-78ff5975c6-fthmt.app-ns-2 Kubernetes SYNCED SYNCED SYNCED SYNCED NOT SENT istiod-usergroup-2-658d6458f7-slpd9 1.17-alpha.f5212a6f7df61fd8156f3585154bed2f003c4117
curl-78ff5975c6-nxtth.app-ns-3 Kubernetes SYNCED SYNCED SYNCED SYNCED NOT SENT istiod-usergroup-2-658d6458f7-slpd9 1.17-alpha.f5212a6f7df61fd8156f3585154bed2f003c4117
{{< /text >}}

### アプリケーション接続が各ユーザーグループ内のみであることの確認{#verify-app-conn-is-only-within-respective-usergroup}

1. `usergroup-1` の `app-ns-1` の `curl` Pod から `usergroup-2` の `app-ns-2` の `httpbin` サービスへリクエストを送信します：

   {{< text bash >}}
   $ kubectl -n app-ns-1 exec "$(kubectl -n app-ns-1 get pod -l app=curl -o jsonpath={.items..metadata.name})" -c curl -- curl -sIL http://httpbin.app-ns-2.svc.cluster.local:8000
   HTTP/1.1 503 Service Unavailable
   content-length: 95
   content-type: text/plain
   date: Sat, 24 Dec 2022 06:54:54 GMT
   server: envoy
   {{< /text >}}

1. `usergroup-2` の `app-ns-2` の `curl` Pod から `usergroup-2` の `app-ns-3` の `httpbin` サービスへリクエストを送信します：通信が成功するはずです：

   {{< text bash >}}
   $ kubectl -n app-ns-2 exec "$(kubectl -n app-ns-2 get pod -l app=curl -o jsonpath={.items..metadata.name})" -c curl -- curl -sIL http://httpbin.app-ns-3.svc.cluster.local:8000
   HTTP/1.1 200 OK
   server: envoy
   date: Thu, 22 Dec 2022 15:01:36 GMT
   content-type: text/html; charset=utf-8
   content-length: 9593
   access-control-allow-origin: \*
   access-control-allow-credentials: true
   x-envoy-upstream-service-time: 3
   {{< /text >}}

## クリーンアップ{#cleanup}

1. 最初のユーザーグループをクリーンアップします：

   {{< text bash >}}
   $ istioctl uninstall --revision usergroup-1 --set values.global.istioNamespace=usergroup-1
   $ kubectl delete ns app-ns-1 usergroup-1
   {{< /text >}}

1. 2 つ目のユーザーグループをクリーンアップします：

   {{< text bash >}}
   $ istioctl uninstall --revision usergroup-2 --set values.global.istioNamespace=usergroup-2
   $ kubectl delete ns app-ns-2 app-ns-3 usergroup-2
   {{< /text >}}

{{< warning >}}
クラスタ管理者は、メッシュ管理者がグローバルな `istioctl uninstall --purge` コマンドを実行できないようにする必要があります。これを実行すると、クラスタ内のすべてのコントロールプレーンがアンインストールされてしまいます。
{{< /warning >}}
