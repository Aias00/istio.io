---
title: Ingress ゲートウェイ
description: Ingress ゲートウェイでアクセス制御を設定する方法を紹介します。
weight: 50
keywords:
  [
    security,
    access-control,
    rbac,
    authorization,
    ingress,
    ip,
    allowlist,
    denylist,
  ]
owner: istio/wg-security-maintainers
test: yes
---

このタスクでは、Istio Ingress ゲートウェイで認可ポリシーを使って IP ベースのアクセス制御を実施する方法を紹介します。

{{< boilerplate gateway-api-support >}}

## 始める前に {#before-you-begin}

このタスクを始める前に、以下を実施してください：

- [Istio 認可の概念](/ja/docs/concepts/security/#authorization)を読んでください。

- [Istio インストールガイド](/ja/docs/setup/install/istioctl/)に従って Istio をインストールしてください。

- Sidecar インジェクションを有効にした名前空間 `foo` に `httpbin` ワークロードをデプロイします：

  {{< text bash >}}
  $ kubectl create ns foo
  $ kubectl label namespace foo istio-injection=enabled
  $ kubectl apply -f @samples/httpbin/httpbin.yaml@ -n foo
  {{< /text >}}

- Ingress ゲートウェイ経由で `httpbin` を公開します：

{{< tabset category-name="config-api" >}}

{{< tab name="Istio APIs" category-value="istio-apis" >}}

ゲートウェイを設定：

{{< text bash >}}
$ kubectl apply -f @samples/httpbin/httpbin-gateway.yaml@ -n foo
{{< /text >}}

Envoy で Ingress ゲートウェイの RBAC デバッグを有効化：

{{< text bash >}}
$ kubectl get pods -n istio-system -o name -l istio=ingressgateway | sed 's|pod/||' | while read -r pod; do istioctl proxy-config log "$pod" -n istio-system --level rbac:debug; done
{{< /text >}}

[Ingress IP とポートの決定](/ja/docs/tasks/traffic-management/ingress/ingress-control/#determining-the-ingress-ip-and-ports)の指示に従い、
`INGRESS_HOST` と `INGRESS_PORT` 環境変数を設定します。

{{< /tab >}}

{{< tab name="Gateway API" category-value="gateway-api" >}}

ゲートウェイを作成：

{{< text bash >}}
$ kubectl apply -f @samples/httpbin/gateway-api/httpbin-gateway.yaml@ -n foo
{{< /text >}}

ゲートウェイの準備を待つ：

{{< text bash >}}
$ kubectl wait --for=condition=programmed gtw -n foo httpbin-gateway
{{< /text >}}

Envoy で Ingress ゲートウェイの RBAC デバッグを有効化：

{{< text bash >}}
$ kubectl get pods -n foo -o name -l gateway.networking.k8s.io/gateway-name=httpbin-gateway | sed 's|pod/||' | while read -r pod; do istioctl proxy-config log "$pod" -n foo --level rbac:debug; done
{{< /text >}}

`INGRESS_PORT` と `INGRESS_HOST` 環境変数を設定：

{{< text bash >}}
$ export INGRESS_HOST=$(kubectl get gtw httpbin-gateway -n foo -o jsonpath='{.status.addresses[0].value}')
$ export INGRESS_PORT=$(kubectl get gtw httpbin-gateway -n foo -o jsonpath='{.spec.listeners[?(@.name=="http")].port}')
{{< /text >}}

{{< /tab >}}

{{< /tabset >}}

- 次のコマンドで `httpbin` ワークロードと Ingress ゲートウェイが正常に動作していることを確認します：

  {{< text bash >}}
  $ curl "$INGRESS_HOST:$INGRESS_PORT"/headers -s -o /dev/null -w "%{http_code}\n"
  200
  {{< /text >}}

  {{< warning >}}
  期待される出力が表示されない場合は、数秒後に再試行してください。
  キャッシュや伝播のオーバーヘッドにより遅延が発生することがあります。
  {{< /warning >}}

## Kubernetes および Istio へのトラフィックの導入 {#getting-traffic-into-Kubernetes-and-Istio}

Kubernetes へのトラフィック導入方法はすべて、すべてのワーカーノードでポートを開くことに関係しています。
主な方法は `NodePort` サービスと `LoadBalancer` サービスです。Kubernetes の `Ingress` リソースも、
Ingress コントローラーによってサポートされている必要があり、そのコントローラーが `NodePort` または `LoadBalancer` サービスを作成します。

- `NodePort` は各ワーカーノードで 30000-32767 の範囲のポートを開き、
  ラベルセレクターでトラフィックを送る Pod を識別します。
  ワーカーノードの前に手動でロードバランサーを作成するか、ラウンドロビン DNS を使う必要があります。

- `LoadBalancer` は `NodePort` と同様ですが、
  環境固有の外部ロードバランサーも作成し、トラフィックをワーカーノードに分配します。
  例えば AWS EKS では、`LoadBalancer` サービスはワーカーノードをターゲットとするクラシック ELB を作成します。
  Kubernetes 環境に `LoadBalancer` 実装がなければ、`NodePort` と同じ動作になります。Istio Ingress ゲートウェイは `LoadBalancer` サービスを作成します。

`NodePort` や `LoadBalancer` からのトラフィックを処理する Pod が、トラフィックを受け取るワーカーノード上で動作していない場合、
Kubernetes の内部プロキシ kube-proxy がパケットを受け取り、正しいノードに転送します。

## オリジナルクライアントのソース IP アドレス {#source-ip-address-of-the-original-client}

パケットが外部プロキシロードバランサーや kube-proxy を経由すると、クライアントの元のソース IP アドレスは失われます。
以下のセクションでは、さまざまなタイプのロードバランサーで元のクライアント IP をログやセキュリティ目的で保持する方法を説明します：

1. [TCP/UDP プロキシロードバランサー](#tcp-proxy)
1. [ネットワークロードバランサー](#network)
1. [HTTP/HTTPS ロードバランサー](#http-https)

以下は、Istio が一般的なマネージド Kubernetes 環境で `LoadBalancer` サービスを使って作成するロードバランサーの種類です：

| クラウドプロバイダー | ロードバランサー名            | ロードバランサータイプ |
| -------------------- | ----------------------------- | ---------------------- |
| AWS EKS              | Classic Elastic Load Balancer | TCP Proxy              |
| GCP GKE              | TCP/UDP Network Load Balancer | Network                |
| Azure AKS            | Azure Load Balancer           | Network                |
| IBM IKS/ROKS         | Network Load Balancer         | Network                |
| DO DOKS              | Load Balancer                 | Network                |

{{< tip >}}
AWS EKS でアノテーション付きで Network Load Balancer を作成するには、ゲートウェイサービスに次のように指定できます：

{{< tabset category-name="config-api" >}}

{{< tab name="Istio APIs" category-value="istio-apis" >}}

{{< text yaml >}}
apiVersion: install.istio.io/v1alpha1
kind: IstioOperator
spec:
meshConfig:
accessLogEncoding: JSON
accessLogFile: /dev/stdout
components:
ingressGateways: - enabled: true
k8s:
hpaSpec:
maxReplicas: 10
minReplicas: 5
serviceAnnotations:
service.beta.kubernetes.io/aws-load-balancer-type: "nlb"
{{< /text >}}

{{< /tab >}}

{{< tab name="Gateway API" category-value="gateway-api" >}}

{{< text yaml >}}
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
name: httpbin-gateway
annotations:
service.beta.kubernetes.io/aws-load-balancer-type: "nlb"
spec:
gatewayClassName: istio
...
{{< /text >}}

{{< /tab >}}

{{< /tabset >}}

{{< /tip >}}

### TCP/UDP プロキシロードバランサー {#tcp-proxy}

TCP/UDP プロキシ外部ロードバランサー（AWS Classic ELB など）を使う場合、
[PROXY プロトコル](https://www.haproxy.com/blog/haproxy/proxy-protocol/)で元のクライアント IP をパケットに埋め込むことができます。
外部ロードバランサーと Istio Ingress ゲートウェイの両方が PROXY プロトコルをサポートしている必要があります。

以下は、PROXY プロトコル対応の AWS EKS で Ingress Gateway をデプロイするサンプル設定です：

{{< tabset category-name="config-api" >}}

{{< tab name="Istio APIs" category-value="istio-apis" >}}

{{< text yaml >}}
apiVersion: install.istio.io/v1alpha1
kind: IstioOperator
spec:
meshConfig:
accessLogEncoding: JSON
accessLogFile: /dev/stdout
defaultConfig:
gatewayTopology:
proxyProtocol: {}
components:
ingressGateways: - enabled: true
name: istio-ingressgateway
k8s:
hpaSpec:
maxReplicas: 10
minReplicas: 5
serviceAnnotations:
service.beta.kubernetes.io/aws-load-balancer-proxy-protocol: "\*"
...
{{< /text >}}

{{< /tab >}}

{{< tab name="Gateway API" category-value="gateway-api" >}}

{{< text yaml >}}
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
name: httpbin-gateway
annotations:
service.beta.kubernetes.io/aws-load-balancer-proxy-protocol: "\*"
proxy.istio.io/config: '{"gatewayTopology" : { "proxyProtocol": {} }}'
spec:
gatewayClassName: istio
...

---

apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
name: httpbin-gateway
spec:
scaleTargetRef:
apiVersion: apps/v1
kind: Deployment
name: httpbin-gateway-istio
minReplicas: 5
maxReplicas: 10
{{< /text >}}

{{< /tab >}}

{{< /tabset >}}

### ネットワークロードバランサー {#network}

クライアント IP を保持する TCP/UDP ネットワークロードバランサー（AWS NLB、GCP 外部 NLB、Azure LB など）やラウンドロビン DNS を使う場合、
kube-proxy をバイパスし、`externalTrafficPolicy: Local` 設定で Kubernetes 内部でもクライアント IP を保持できます。

{{< warning >}}
本番環境で `externalTrafficPolicy: Local` を有効にする場合は、**Ingress ゲートウェイインスタンスを複数ノードにデプロイすることを強く推奨します**。
そうしないと、Ingress ゲートウェイインスタンスが動作しているノードだけが NLB トラフィックを受け付け、
クラスタ全体への Ingress トラフィックがボトルネックになったり、ノード障害時に完全に Ingress トラフィックが失われる可能性があります。
詳細は[サービスソース IP `Type=NodePort`](https://kubernetes.io/ja/docs/tutorials/services/source-ip/#source-ip-for-services-with-type-nodeport)を参照してください。
{{< /warning >}}

Ingress ゲートウェイで元のクライアント IP を保持するには、次のコマンドで `externalTrafficPolicy: Local` を設定します：

{{< tabset category-name="config-api" >}}

{{< tab name="Istio APIs" category-value="istio-apis" >}}

{{< text bash >}}
$ kubectl patch svc istio-ingressgateway -n istio-system -p '{"spec":{"externalTrafficPolicy":"Local"}}'
{{< /text >}}

{{< /tab >}}

{{< tab name="Gateway API" category-value="gateway-api" >}}

{{< text bash >}}
$ kubectl patch svc httpbin-gateway-istio -n foo -p '{"spec":{"externalTrafficPolicy":"Local"}}'
{{< /text >}}

{{< /tab >}}

{{< /tabset >}}

### HTTP/HTTPS ロードバランサー {#http-https}

HTTP/HTTPS 外部ロードバランサー（AWS、ALB、GCP など）を使う場合、元のクライアント IP は X-Forwarded-For ヘッダーに格納されます。
Istio でこのヘッダーからクライアント IP を抽出するには追加設定が必要です。
[ゲートウェイネットワークトポロジーの設定](/ja/docs/ops/configuration/traffic-management/network-topologies/)を参照してください。
Kubernetes の前に単一のロードバランサーを使う場合の簡単な例：

{{< text yaml >}}
apiVersion: install.istio.io/v1alpha1
kind: IstioOperator
spec:
meshConfig:
accessLogEncoding: JSON
accessLogFile: /dev/stdout
defaultConfig:
gatewayTopology:
numTrustedProxies: 1
{{< /text >}}

## IP ベースの許可リスト・拒否リスト {#ip-based-allow-list-and-deny-list}

**`ipBlocks` と `remoteIpBlocks` の使い分け:** X-Forwarded-For HTTP ヘッダーや PROXY プロトコルで元のクライアント IP を判定する場合は、`AuthorizationPolicy` で `remoteIpBlocks` を使います。
`externalTrafficPolicy: Local` を使う場合は `ipBlocks` を使ってください。

| ロードバランサータイプ | クライアントソース IP | `ipBlocks` と `remoteIpBlocks` |
| ---------------------- | --------------------- | ------------------------------ |
| TCP Proxy              | PROXY Protocol        | `remoteIpBlocks`               |
| Network                | packet source address | `ipBlocks`                     |
| HTTP/HTTPS             | X-Forwarded-For       | `remoteIpBlocks`               |

- 次のコマンドで Istio Ingress ゲートウェイに認可ポリシー `ingress-policy` を作成します。
  このポリシーは `action` フィールドを `ALLOW` に設定し、`ipBlocks` で指定した IP アドレスからのアクセスのみを許可します。
  リストにない IP アドレスは拒否されます。`ipBlocks` は単一 IP や CIDR 表記をサポートします。

{{< tabset category-name="config-api" >}}

{{< tab name="Istio APIs" category-value="istio-apis" >}}

**_ipBlocks:_**

{{< text bash >}}
$ kubectl apply -f - <<EOF
apiVersion: security.istio.io/v1
kind: AuthorizationPolicy
metadata:
name: ingress-policy
namespace: istio-system
spec:
selector:
matchLabels:
app: istio-ingressgateway
action: ALLOW
rules:

- from: - source:
  ipBlocks: ["1.2.3.4", "5.6.7.0/24"]
  EOF
  {{< /text >}}

**_remoteIpBlocks:_**

{{< text bash >}}
$ kubectl apply -f - <<EOF
apiVersion: security.istio.io/v1
kind: AuthorizationPolicy
metadata:
name: ingress-policy
namespace: istio-system
spec:
selector:
matchLabels:
app: istio-ingressgateway
action: ALLOW
rules:

- from: - source:
  remoteIpBlocks: ["1.2.3.4", "5.6.7.0/24"]
  EOF
  {{< /text >}}

{{< /tab >}}

{{< tab name="Gateway API" category-value="gateway-api" >}}

**_ipBlocks:_**

{{< text bash >}}
$ kubectl apply -f - <<EOF
apiVersion: security.istio.io/v1
kind: AuthorizationPolicy
metadata:
name: ingress-policy
namespace: foo
spec:
targetRef:
kind: Gateway
group: gateway.networking.k8s.io
name: httpbin-gateway
action: ALLOW
rules:

- from: - source:
  ipBlocks: ["1.2.3.4", "5.6.7.0/24"]
  EOF
  {{< /text >}}

**_remoteIpBlocks:_**

{{< text bash >}}
$ kubectl apply -f - <<EOF
apiVersion: security.istio.io/v1
kind: AuthorizationPolicy
metadata:
name: ingress-policy
namespace: foo
spec:
targetRef:
kind: Gateway
group: gateway.networking.k8s.io
name: httpbin-gateway
action: ALLOW
rules:

- from: - source:
  remoteIpBlocks: ["1.2.3.4", "5.6.7.0/24"]
  EOF
  {{< /text >}}

{{< /tab >}}

{{< /tabset >}}

- 次のコマンドで Ingress ゲートウェイへのリクエストが拒否されることを確認します：

  {{< text bash >}}
  $ curl "$INGRESS_HOST:$INGRESS_PORT"/headers -s -o /dev/null -w "%{http_code}\n"
  403
  {{< /text >}}

- 元のクライアント IP アドレスを環境変数に設定します。分からない場合は、次のコマンドで Envoy ログから取得できます：

{{< tabset category-name="config-api" >}}

{{< tab name="Istio APIs" category-value="istio-apis" >}}

**_ipBlocks:_**

{{< text bash >}}
$ CLIENT_IP=$(kubectl get pods -n istio-system -o name -l istio=ingressgateway | sed 's|pod/||' | while read -r pod; do kubectl logs "$pod" -n istio-system | grep remoteIP; done | tail -1 | awk -F, '{print $3}' | awk -F: '{print $2}' | sed 's/ //') && echo "$CLIENT_IP"
192.168.10.15
{{< /text >}}

**_remoteIpBlocks:_**

{{< text bash >}}
$ CLIENT_IP=$(kubectl get pods -n istio-system -o name -l istio=ingressgateway | sed 's|pod/||' | while read -r pod; do kubectl logs "$pod" -n istio-system | grep remoteIP; done | tail -1 | awk -F, '{print $4}' | awk -F: '{print $2}' | sed 's/ //') && echo "$CLIENT_IP"
192.168.10.15
{{< /text >}}

{{< /tab >}}

{{< tab name="Gateway API" category-value="gateway-api" >}}

**_ipBlocks:_**

{{< text bash >}}
$ CLIENT_IP=$(kubectl get pods -n foo -o name -l gateway.networking.k8s.io/gateway-name=httpbin-gateway | sed 's|pod/||' | while read -r pod; do kubectl logs "$pod" -n foo | grep remoteIP; done | tail -1 | awk -F, '{print $3}' | awk -F: '{print $2}' | sed 's/ //') && echo "$CLIENT_IP"
192.168.10.15
{{< /text >}}

**_remoteIpBlocks:_**

{{< text bash >}}
$ CLIENT_IP=$(kubectl get pods -n foo -o name -l gateway.networking.k8s.io/gateway-name=httpbin-gateway | sed 's|pod/||' | while read -r pod; do kubectl logs "$pod" -n foo | grep remoteIP; done | tail -1 | awk -F, '{print $4}' | awk -F: '{print $2}' | sed 's/ //') && echo "$CLIENT_IP"
192.168.10.15
{{< /text >}}

{{< /tab >}}

{{< /tabset >}}

- `ingress-policy` を更新してクライアント IP アドレスを含めます：

{{< tabset category-name="config-api" >}}

{{< tab name="Istio APIs" category-value="istio-apis" >}}

**_ipBlocks:_**

{{< text bash >}}
$ kubectl apply -f - <<EOF
apiVersion: security.istio.io/v1
kind: AuthorizationPolicy
metadata:
name: ingress-policy
namespace: istio-system
spec:
selector:
matchLabels:
app: istio-ingressgateway
action: ALLOW
rules:

- from: - source:
  ipBlocks: ["1.2.3.4", "5.6.7.0/24", "$CLIENT_IP"]
  EOF
  {{< /text >}}

**_remoteIpBlocks:_**

{{< text bash >}}
$ kubectl apply -f - <<EOF
apiVersion: security.istio.io/v1
kind: AuthorizationPolicy
metadata:
name: ingress-policy
namespace: istio-system
spec:
selector:
matchLabels:
app: istio-ingressgateway
action: ALLOW
rules:

- from: - source:
  remoteIpBlocks: ["1.2.3.4", "5.6.7.0/24", "$CLIENT_IP"]
  EOF
  {{< /text >}}

{{< /tab >}}

{{< tab name="Gateway API" category-value="gateway-api" >}}

**_ipBlocks:_**

{{< text bash >}}
$ kubectl apply -f - <<EOF
apiVersion: security.istio.io/v1
kind: AuthorizationPolicy
metadata:
name: ingress-policy
namespace: foo
spec:
targetRef:
kind: Gateway
group: gateway.networking.k8s.io
name: httpbin-gateway
action: ALLOW
rules:

- from: - source:
  ipBlocks: ["1.2.3.4", "5.6.7.0/24", "$CLIENT_IP"]
  EOF
  {{< /text >}}

**_remoteIpBlocks:_**

{{< text bash >}}
$ kubectl apply -f - <<EOF
apiVersion: security.istio.io/v1
kind: AuthorizationPolicy
metadata:
name: ingress-policy
namespace: foo
spec:
targetRef:
kind: Gateway
group: gateway.networking.k8s.io
name: httpbin-gateway
action: ALLOW
rules:

- from: - source:
  remoteIpBlocks: ["1.2.3.4", "5.6.7.0/24", "$CLIENT_IP"]
  EOF
  {{< /text >}}

{{< /tab >}}

{{< /tabset >}}

- Ingress ゲートウェイへのリクエストが許可されることを確認します：

  {{< text bash >}}
  $ curl "$INGRESS_HOST:$INGRESS_PORT"/headers -s -o /dev/null -w "%{http_code}\n"
  200
  {{< /text >}}

- `ingress-policy` 認可ポリシーを更新し、`action` キーを `DENY` に設定して、
  `ipBlocks` で指定した IP アドレスからの Ingress ゲートウェイへのアクセスを拒否します：

{{< tabset category-name="config-api" >}}

{{< tab name="Istio APIs" category-value="istio-apis" >}}

**_ipBlocks:_**

{{< text bash >}}
$ kubectl apply -f - <<EOF
apiVersion: security.istio.io/v1
kind: AuthorizationPolicy
metadata:
name: ingress-policy
namespace: istio-system
spec:
selector:
matchLabels:
app: istio-ingressgateway
action: DENY
rules:

- from: - source:
  ipBlocks: ["$CLIENT_IP"]
  EOF
  {{< /text >}}

**_remoteIpBlocks:_**

{{< text bash >}}
$ kubectl apply -f - <<EOF
apiVersion: security.istio.io/v1
kind: AuthorizationPolicy
metadata:
name: ingress-policy
namespace: istio-system
spec:
selector:
matchLabels:
app: istio-ingressgateway
action: DENY
rules:

- from: - source:
  remoteIpBlocks: ["$CLIENT_IP"]
  EOF
  {{< /text >}}

{{< /tab >}}

{{< tab name="Gateway API" category-value="gateway-api" >}}

**_ipBlocks:_**

{{< text bash >}}
$ kubectl apply -f - <<EOF
apiVersion: security.istio.io/v1
kind: AuthorizationPolicy
metadata:
name: ingress-policy
namespace: foo
spec:
targetRef:
kind: Gateway
group: gateway.networking.k8s.io
name: httpbin-gateway
action: DENY
rules:

- from: - source:
  ipBlocks: ["$CLIENT_IP"]
  EOF
  {{< /text >}}

**_remoteIpBlocks:_**

{{< text bash >}}
$ kubectl apply -f - <<EOF
apiVersion: security.istio.io/v1
kind: AuthorizationPolicy
metadata:
name: ingress-policy
namespace: foo
spec:
targetRef:
kind: Gateway
group: gateway.networking.k8s.io
name: httpbin-gateway
action: DENY
rules:

- from: - source:
  remoteIpBlocks: ["$CLIENT_IP"]
  EOF
  {{< /text >}}

{{< /tab >}}

{{< /tabset >}}

- Ingress ゲートウェイへのリクエストが拒否されることを確認します：

  {{< text bash >}}
  $ curl "$INGRESS_HOST:$INGRESS_PORT"/headers -s -o /dev/null -w "%{http_code}\n"
  403
  {{< /text >}}

- オンラインプロキシサービスを使って、異なるクライアント IP で Ingress ゲートウェイにアクセスし、リクエストが許可されるかどうかを検証できます。

- 期待される応答が得られない場合は、Ingress ゲートウェイのログで RBAC デバッグ情報を確認してください：

{{< tabset category-name="config-api" >}}

{{< tab name="Istio APIs" category-value="istio-apis" >}}

{{< text bash >}}
$ kubectl get pods -n istio-system -o name -l istio=ingressgateway | sed 's|pod/||' | while read -r pod; do kubectl logs "$pod" -n istio-system; done
{{< /text >}}

{{< /tab >}}

{{< tab name="Gateway API" category-value="gateway-api" >}}

{{< text bash >}}
$ kubectl get pods -n foo -o name -l gateway.networking.k8s.io/gateway-name=httpbin-gateway | sed 's|pod/||' | while read -r pod; do kubectl logs "$pod" -n foo; done
{{< /text >}}

{{< /tab >}}

{{< /tabset >}}

## クリーンアップ {#clean-up}

- 認可ポリシーを削除します：

{{< tabset category-name="config-api" >}}

{{< tab name="Istio APIs" category-value="istio-apis" >}}

{{< text bash >}}
$ kubectl delete authorizationpolicy ingress-policy -n istio-system
{{< /text >}}

{{< /tab >}}

{{< tab name="Gateway API" category-value="gateway-api" >}}

{{< text bash >}}
$ kubectl delete authorizationpolicy ingress-policy -n foo
{{< /text >}}

{{< /tab >}}

{{< /tabset >}}

- 名前空間 `foo` を削除します：

  {{< text bash >}}
  $ kubectl delete namespace foo
  {{< /text >}}
