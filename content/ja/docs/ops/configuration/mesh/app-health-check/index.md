---
title: Istio サービスのヘルスチェック
description: Istio サービスのヘルスチェック方法を紹介します。
weight: 50
aliases:
  - /zh/docs/tasks/traffic-management/app-health-check/
  - /zh/docs/ops/security/health-checks-and-mtls/
  - /zh/help/ops/setup/app-health-check
  - /zh/help/ops/app-health-check
  - /zh/docs/ops/app-health-check
  - /zh/docs/ops/setup/app-health-check
keywords: [security, health-check]
owner: istio/wg-user-experience-maintainers
test: yes
---

[Kubernetes の Liveness/Readiness/Startup プローブ](https://kubernetes.io/ja/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/)
では、以下のようなプローブの設定方法が説明されています：

1. [コマンド方式](https://kubernetes.io/ja/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/#define-a-liveness-command)
1. [HTTP リクエスト](https://kubernetes.io/ja/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/#define-a-liveness-http-request)
1. [TCP プローブ](https://kubernetes.io/ja/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/#define-a-tcp-liveness-probe)
1. [gRPC プローブ](https://kubernetes.io/ja/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/#define-a-grpc-liveness-probe)

コマンド方式はそのまま動作しますが、HTTP リクエスト、TCP プローブ、gRPC プローブは Istio による Pod 設定の変更が必要です。

`liveness-http` サービスへのヘルスチェックリクエストは kubelet から送信されます。双方向 TLS を有効にしている場合、
kubelet は Istio が発行した証明書を持っていないため、
ヘルスチェックリクエストが失敗します。

TCP プローブは特別な扱いが必要です。Istio はすべての受信トラフィックを Sidecar にリダイレクトするため、
すべての TCP ポートがオープンに見えます。kubelet は指定ポートでプロセスがリッスンしているかだけを確認するため、
Sidecar が動作していればこのプローブは常に成功します。

Istio はアプリケーションの `PodSpec` の Readiness/Liveness プローブをリライトし、
[Sidecar プロキシ](/ja/docs/reference/commands/pilot-agent/) にプローブリクエストを転送することで、これらの問題を解決します。

## Liveness プローブのリライト例 {#liveness-probe-rewrite-example}

アプリケーションの `PodSpec` レベルで Liveness/Readiness プローブがどのようにリライトされるかを示すため、
[liveness-http-same-port サンプル]({{< github_file >}}/samples/health-check/liveness-http-same-port.yaml)を利用できます。

まず、このサンプル用に名前空間を作成し、ラベルを付与します：

{{< text bash >}}
$ kubectl create namespace istio-io-health-rewrite
$ kubectl label namespace istio-io-health-rewrite istio-injection=enabled
{{< /text >}}

次にサンプルアプリケーションをデプロイします：

{{< text bash yaml >}}
$ kubectl apply -f - <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
name: liveness-http
namespace: istio-io-health-rewrite
spec:
selector:
matchLabels:
app: liveness-http
version: v1
template:
metadata:
labels:
app: liveness-http
version: v1
spec:
containers: - name: liveness-http
image: docker.io/istio/health:example
ports: - containerPort: 8001
livenessProbe:
httpGet:
path: /foo
port: 8001
initialDelaySeconds: 5
periodSeconds: 5
EOF
{{< /text >}}

デプロイ後、Pod のアプリケーションコンテナを確認すると、パスが変更されていることが分かります：

{{< text bash json >}}
$ kubectl get pod "$LIVENESS_POD" -n istio-io-health-rewrite -o json | jq '.spec.containers[0].livenessProbe.httpGet'
{
"path": "/app-health/liveness-http/livez",
"port": 15020,
"scheme": "HTTP"
}
{{< /text >}}

元の `livenessProbe` のパスは、Sidecar コンテナの環境変数 `ISTIO_KUBE_APP_PROBERS` の新しいパスにマッピングされています：

{{< text bash json >}}
$ kubectl get pod "$LIVENESS_POD" -n istio-io-health-rewrite -o=jsonpath="{.spec.containers[1].env[?(@.name=='ISTIO_KUBE_APP_PROBERS')]}"
{
"name":"ISTIO_KUBE_APP_PROBERS",
"value":"{\"/app-health/liveness-http/livez\":{\"httpGet\":{\"path\":\"/foo\",\"port\":8001,\"scheme\":\"HTTP\"},\"timeoutSeconds\":1}}"
}
{{< /text >}}

HTTP や gRPC リクエストの場合、Sidecar プロキシはリクエストをアプリケーションに転送し、レスポンスボディを除去してステータスコードのみ返します。
TCP プローブの場合、Sidecar プロキシはトラフィックリダイレクトを回避しつつポートチェックを行います。

すべての組み込み Istio [プロファイル](/ja/docs/setup/additional-setup/config-profiles/) で、
問題のあるプローブのリライトはデフォルトで有効ですが、後述の方法で無効化できます。

## コマンド方式の Liveness/Readiness プローブ {#liveness-and-readiness-probes-using-the-command-approach}

Istio では[コマンド方式の Liveness サンプル]({{< github_file >}}/samples/health-check/liveness-command.yaml)も提供しています。
このプローブが双方向 TLS 有効時にどのように動作するかを示すため、まず名前空間を作成します：

{{< text bash >}}
$ kubectl create ns istio-io-health
{{< /text >}}

`STRICT` モードの双方向 TLS を設定するには、次のコマンドを実行します：

{{< text bash >}}
$ kubectl apply -f - <<EOF
apiVersion: security.istio.io/v1
kind: PeerAuthentication
metadata:
name: "default"
namespace: "istio-io-health"
spec:
mtls:
mode: STRICT
EOF
{{< /text >}}

次に、以下のコマンドでサンプルサービスをデプロイします：

{{< text bash >}}
$ kubectl -n istio-io-health apply -f <(istioctl kube-inject -f @samples/health-check/liveness-command.yaml@)
{{< /text >}}

Liveness プローブが正常に動作しているか確認するには、サンプル Pod の状態を確認し、実行中であることを検証します。

{{< text bash >}}
$ kubectl -n istio-io-health get pod
NAME READY STATUS RESTARTS AGE
liveness-6857c8775f-zdv9r 2/2 Running 0 4m
{{< /text >}}

## HTTP、TCP、gRPC 方式の Liveness/Readiness プローブ {#liveness-and-readiness-probes-using-the-http-request-approach}

前述の通り、Istio はデフォルトでプローブリライトを使って HTTP、TCP、gRPC プローブを実現します。
この機能は特定の Pod またはグローバルで無効化できます。

### Pod 単位でプローブリライトを無効化 {#disable-the-http-probe-rewrite-for-a-pod}

`sidecar.istio.io/rewriteAppHTTPProbers: "false"` を使って
[Pod にアノテーションを追加](/ja/docs/reference/config/annotations/)することで、プローブリライトを無効化できます。
このアノテーションは必ず [Pod リソース](https://kubernetes.io/ja/docs/concepts/workloads/pods/pod-overview/) に追加してください。
（Deployment リソースなど他の場所では無視されます）

{{< tabset category-name="disable-probe-rewrite" >}}

{{< tab name="HTTP Probe" category-value="http-probe" >}}

{{< text yaml >}}
kubectl apply -f - <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
name: liveness-http
spec:
selector:
matchLabels:
app: liveness-http
version: v1
template:
metadata:
labels:
app: liveness-http
version: v1
annotations:
sidecar.istio.io/rewriteAppHTTPProbers: "false"
spec:
containers: - name: liveness-http
image: docker.io/istio/health:example
ports: - containerPort: 8001
livenessProbe:
httpGet:
path: /foo
port: 8001
initialDelaySeconds: 5
periodSeconds: 5
EOF
{{< /text >}}

{{< /tab >}}

{{< tab name="gRPC Probe" category-value="grpc-probe" >}}

{{< text yaml >}}
kubectl apply -f - <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
name: liveness-grpc
spec:
selector:
matchLabels:
app: liveness-grpc
version: v1
template:
metadata:
labels:
app: liveness-grpc
version: v1
annotations:
sidecar.istio.io/rewriteAppHTTPProbers: "false"
spec:
containers: - name: etcd
image: registry.k8s.io/etcd:3.5.1-0
command: ["--listen-client-urls", "http://0.0.0.0:2379", "--advertise-client-urls", "http://127.0.0.1:2379", "--log-level", "debug"]
ports: - containerPort: 2379
livenessProbe:
grpc:
port: 2379
initialDelaySeconds: 10
periodSeconds: 5
EOF
{{< /text >}}

{{< /tab >}}

{{< /tabset >}}

この方法により、Istio を再インストールせずに単一の Deployment ごとにヘルスチェックプローブのリライトを段階的に無効化できます。

### グローバルにプローブリライトを無効化 {#disable-the-probe-rewrite-globally}

[Istio のインストール](/ja/docs/setup/install/istioctl/)時に
`--set values.sidecarInjectorWebhook.rewriteAppHTTPProbe=false`
でグローバルにプローブリライトを無効化できます。**または** Istio Sidecar インジェクターの ConfigMap を更新します：

{{< text bash >}}
$ kubectl get cm istio-sidecar-injector -n istio-system -o yaml | sed -e 's/"rewriteAppHTTPProbe": true/"rewriteAppHTTPProbe": false/' | kubectl apply -f -
{{< /text >}}

## クリーンアップ {#cleanup}

これらのサンプルで使った名前空間を削除します：

{{< text bash >}}
$ kubectl delete ns istio-io-health istio-io-health-rewrite
{{< /text >}}
