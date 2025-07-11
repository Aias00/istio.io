---
title: 外部コントロールプレーンを使用した Istio のインストール
description: 外部コントロールプレーンとリモートクラスターのインストール。
weight: 80
aliases:
  - /ja/docs/setup/additional-setup/external-controlplane/
  - /latest/ja/docs/setup/additional-setup/external-controlplane/
keywords: [external, control, istiod, remote]
owner: istio/wg-environments-maintainers
test: no
---

このガイドでは、{{< gloss "external control plane">}}外部コントロールプレーン{{< /gloss >}}をインストールし、
その後 1 つ以上の{{< gloss "remote cluster" >}}リモートクラスター{{< /gloss >}}をそのプレーンに接続するプロセスを説明します。

外部コントロールプレーンの[デプロイメントモデル](/ja/docs/ops/deployment/deployment-models/#control-plane-models)
により、メッシュオペレーターは、メッシュを構成するデータプレーンクラスター（または複数のクラスター）とは別の外部クラスターでコントロールプレーンをインストールおよび管理できます。
このデプロイメントモデルにより、メッシュオペレーターとメッシュ管理者を明確に区別できます。メッシュオペレーターは Istio コントロールプレーンをインストールおよび管理し、
メッシュ管理者はメッシュの設定のみを行います。

{{< image width="75%"
    link="external-controlplane.svg"
    caption="外部コントロールプレーンクラスターとリモートクラスター"
    >}}

リモートクラスターで実行されている Envoy プロキシ（Sidecar と Gateway）は、Ingress Gateway を介して外部 Istiod にアクセスし、
検出、CA、注入、検証のためのエンドポイントを外部に公開します。

外部コントロールプレーンの設定と管理は、外部クラスター内のメッシュオペレーターが行いますが、
最初のリモートクラスターは、メッシュ自体の設定クラスターとして機能します。メッシュサービス自体に加えて、
メッシュ管理者は設定クラスターを使用してメッシュリソース（Gateway、仮想サービスなど）を設定します。
外部コントロールプレーンは、上図に示すように、Kubernetes API Server からリモートでこの設定にアクセスします。

## 始める前に {#before-you-begin}

### クラスター {#clusters}

このガイドでは、任意の 2 つの[サポートされているバージョンの Kubernetes](/ja/docs/releases/supported-releases#support-status-of-istio-releases)
クラスターが必要です：{{< supported_kubernetes_versions >}}。

最初のクラスターは、`external-istiod` 名前空間にインストールされた{{< gloss "external control plane">}}外部コントロールプレーン{{< /gloss >}}をホストします。
Ingress Gateway も `istio-system` 名前空間にインストールされており、外部コントロールプレーンへのクラスター間アクセスを提供します。

2 番目のクラスターは、{{< gloss "remote cluster">}}リモートクラスター{{< /gloss >}}で、メッシュアプリケーションワークロードを実行します。
その Kubernetes API Server は、外部コントロールプレーン（Istiod）がワークロードプロキシを設定するために使用するメッシュ設定を提供します。

### API Server アクセス {#API-server-access}

外部コントロールプレーンクラスターは、リモートクラスターの Kubernetes API Server にアクセスできる必要があります。
多くのクラウドプロバイダーは、ネットワーク負荷分散器（NLB）を介して API Server へのアクセスを公開しています。
API Server に直接アクセスできない場合は、アクセスを有効にするためにインストールプロセスを変更する必要があります。
たとえば、[マルチクラスター設定](#adding-clusters)で使用される[東西トラフィック](https://en.wikipedia.org/wiki/East-west_traffic)
Gateway も、API Server へのアクセスを有効にするために使用できます。

### 環境変数 {#environment-variables}

以下環境変数は、説明を簡略化するために使用されます：

| 変数名                 | 説明                                                                                                                                                                                                                             |
| ---------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `CTX_EXTERNAL_CLUSTER` | デフォルトの [Kubernetes 設定ファイル](https://kubernetes.io/ja/docs/tasks/access-application-cluster/configure-access-multiple-clusters/)のコンテキスト名。外部コントロールプレーンクラスターにアクセスするために使用されます。 |
| `CTX_REMOTE_CLUSTER`   | デフォルトの [Kubernetes 設定ファイル](https://kubernetes.io/ja/docs/tasks/access-application-cluster/configure-access-multiple-clusters/)のコンテキスト名。リモートクラスターにアクセスするために使用されます。                 |
| `REMOTE_CLUSTER_NAME`  | リモートクラスターの名前。                                                                                                                                                                                                       |
| `EXTERNAL_ISTIOD_ADDR` | 外部コントロールプレーンクラスター上の Ingress Gateway のホスト名。リモートクラスターはこれを使用して外部コントロールプレーンにアクセスします。                                                                                  |
| `SSL_SECRET_NAME`      | 外部コントロールプレーンクラスター上の Ingress Gateway の TLS 証明書を持つ Secret の名前。                                                                                                                                       |

`CTX_EXTERNAL_CLUSTER`、`CTX_REMOTE_CLUSTER`、および `REMOTE_CLUSTER_NAME` を

{{< text syntax=bash snip_id=none >}}
$ export CTX_EXTERNAL_CLUSTER=<外部クラスターのコンテキスト>
$ export CTX_REMOTE_CLUSTER=<リモートクラスターのコンテキスト>
$ export REMOTE_CLUSTER_NAME=<リモートクラスターの名前>
{{< /text >}}

## クラスター設定 {#cluster-configuration}

### メッシュオペレーターの手順 {#mesh-operator-steps}

メッシュオペレーターは、外部クラスター上で外部 Istio コントロールプレーンをインストールおよび管理します。
これには、外部クラスター上で Ingress Gateway を構成し、リモートクラスターがコントロールプレーンにアクセスできるようにし、
外部コントロールプレーンを使用するために必要な Webhook、ConfigMap、および Secret をリモートクラスターにインストールすることが含まれます。

#### 外部クラスターで Gateway をセットアップする {#set-up-a-gateway-in-the-external-cluster}

1. Ingress Gateway の Istio インストール設定を作成し、外部コントロールプレーンのポートを他のクラスターに公開します：

   {{< text bash >}}
   $ cat <<EOF > controlplane-gateway.yaml
   apiVersion: install.istio.io/v1alpha1
   kind: IstioOperator
   metadata:
   namespace: istio-system
   spec:
   components:
   ingressGateways: - name: istio-ingressgateway
   enabled: true
   k8s:
   service:
   ports: - port: 15021
   targetPort: 15021
   name: status-port - port: 15012
   targetPort: 15012
   name: tls-xds - port: 15017
   targetPort: 15017
   name: tls-webhook
   EOF
   {{< /text >}}

   次に、外部クラスターの `istio-system` 名前空間に Gateway をインストールします：

   {{< text bash >}}
   $ istioctl install -f controlplane-gateway.yaml --context="${CTX_EXTERNAL_CLUSTER}"
   {{< /text >}}

1. 以下のコマンドを実行して、Ingress Gateway が起動して実行中であることを

   {{< text bash >}}
   $ kubectl get po -n istio-system --context="${CTX_EXTERNAL_CLUSTER}"
   NAME READY STATUS RESTARTS AGE
   istio-ingressgateway-9d4c7f5c7-7qpzz 1/1 Running 0 29s
   istiod-68488cd797-mq8dn 1/1 Running 0 38s
   {{< /text >}}

   `istio-system` 名前空間にも Istiod Deployment が作成されていることに注意してください。
   これは、リモートクラスターで使用されるコントロールプレーンを構成するために使用されます。

   {{< tip >}}
   Ingress Gateway を外部クラスター上の異なる名前空間で複数の外部コントロールプレーンをホストするように構成できますが、
   本例では、外部 Istiod を `external-istiod` 名前空間にのみデプロイします。
   {{< /tip >}}

1. 外部コントロールプレーンの Ingress Gateway サービスを公開するために、TLS を使用したパブリックホスト名で環境を構成します。

   `EXTERNAL_ISTIOD_ADDR` 環境変数をホスト名に設定し、`SSL_SECRET_NAME` 環境変数を TLS 証明書を含む Secret に設定します：

   {{< text syntax=bash snip_id=none >}}
   $ export EXTERNAL_ISTIOD_ADDR=<外部 istiod ホスト>
   $ export SSL_SECRET_NAME=<外部 istiod secret>
   {{< /text >}}

   これらの説明は、外部クラスターの Gateway を公開するために、正しい署名付き DNS 証明書を使用したホスト名を使用することを前提としています。
   これは、本番環境で推奨される方法です。
   [安全な Ingress タスク](/ja/docs/tasks/traffic-management/ingress/secure-ingress/#configure-a-tls-ingress-gateway-for-a-single-host)を参照してください。

   環境変数は次のようになります：

   {{< text bash >}}
   $ echo "$EXTERNAL_ISTIOD_ADDR" "$SSL_SECRET_NAME"
   myhost.example.com myhost-example-credential
   {{< /text >}}

   {{< tip >}}
   DNS ホスト名がないが、テスト環境で外部コントロールプレーンを試す場合は、外部ロードバランサーの IP アドレスを使用して Gateway にアクセスできます：

   {{< text bash >}}
   $ export EXTERNAL_ISTIOD_ADDR=$(kubectl -n istio-system --context="${CTX_EXTERNAL_CLUSTER}" get svc istio-ingressgateway -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
   $ export SSL_SECRET_NAME=NONE
   {{< /text >}}

   これには、設定にいくつかの変更が必要です。以下の説明のすべての関連手順に従ってください。
   {{< /tip >}}

#### リモートクラスターの設定 {#set-up-the-remote-cluster}

1. `remote` 設定ファイルを使用して、リモートクラスター上で Istio をインストールします。
   これにより、外部コントロールプレーンの注入 Webhook を使用する注入 Webhook がインストールされます。
   これは、ローカルでデプロイされた注入器ではなく、このクラスターは設定クラスターとしても機能するためです。
   したがって、リモートクラスター上で Istio CRD およびその他のリソースをインストールする際に、
   `global.configCluster` と `pilot.configMap` を `true` に設定します：

   {{< text syntax=bash snip_id=get_remote_config_cluster_iop >}}
   $ cat <<EOF > remote-config-cluster.yaml
   apiVersion: install.istio.io/v1alpha1
   kind: IstioOperator
   metadata:
   namespace: external-istiod
   spec:
   profile: remote
   values:
   global:
   istioNamespace: external-istiod
   configCluster: true
   pilot:
   configMap: true
   istiodRemote:
   injectionURL: https://${EXTERNAL_ISTIOD_ADDR}:15017/inject/cluster/${REMOTE_CLUSTER_NAME}/net/network1
   base:
   validationURL: https://${EXTERNAL_ISTIOD_ADDR}:15017/validate
   EOF
   {{< /text >}}

   {{< tip >}}
   クラスター名に `/`（スラッシュ）文字が含まれている場合は、`injectionURL` で `--slash--` に置き換えてください。
   たとえば、`injectionURL: https://1.2.3.4:15017/inject/cluster/`<mark>`cluster--slash--1`</mark>`/net/network1`。
   {{< /tip >}}

1. `EXTERNAL_ISTIOD_ADDR` が IP アドレスではなく、正しい DNS ホスト名を使用している場合は、
   検出アドレスとパスを指定するように設定を変更します：

   {{< warning >}}
   本番環境では、これを行うことはお勧めしません。
   {{< /warning >}}

   {{< text bash >}}
   $ sed -i'.bk' \
    -e "s|injectionURL: https://${EXTERNAL_ISTIOD_ADDR}:15017|injectionPath: |" \
    -e "/istioNamespace:/a\\
   remotePilotAddress: ${EXTERNAL_ISTIOD_ADDR}" \
    -e '/base:/,+1d' \
    remote-config-cluster.yaml; rm remote-config-cluster.yaml.bk
   {{< /text >}}

1. リモートクラスター上で設定をインストールします：

   {{< text bash >}}
   $ kubectl create namespace external-istiod --context="${CTX_REMOTE_CLUSTER}"
    $ istioctl install -f remote-config-cluster.yaml --set values.defaultRevision=default --context="${CTX_REMOTE_CLUSTER}"
   {{< /text >}}

1. リモートクラスターの注入 Webhook 設定がインストールされていることを確認します：

   {{< text bash >}}
   $ kubectl get mutatingwebhookconfiguration --context="${CTX_REMOTE_CLUSTER}"
   NAME WEBHOOKS AGE
   istio-revision-tag-default-external-istiod 4 2m2s
   istio-sidecar-injector-external-istiod 4 2m5s
   {{< /text >}}

1. リモートクラスターの検証 Webhook 設定がインストールされていることを確認します：

   {{< text bash >}}
   $ kubectl get validatingwebhookconfiguration --context="${CTX_REMOTE_CLUSTER}"
   NAME WEBHOOKS AGE
   istio-validator-external-istiod 1 6m53s
   istiod-default-validator 1 6m53s
   {{< /text >}}

#### 外部クラスターでコントロールプレーンをインストールする {#set-up-the-control-plane-in-the-external-cluster}

1. 外部コントロールプレーンをホストするために `external-istiod` 名前空間を作成します：

   {{< text bash >}}
   $ kubectl create namespace external-istiod --context="${CTX_EXTERNAL_CLUSTER}"
   {{< /text >}}

1. 外部コントロールプレーンは、リモートクラスターからサービス、端点、および Pod 属性を検出する必要があります。
   資格情報を持つ Secret を作成し、リモートクラスターの `kube-apiserver` にアクセスし、
   外部クラスターにインストールします：

   {{< text bash >}}
   $ istioctl create-remote-secret \
    --context="${CTX_REMOTE_CLUSTER}" \
      --type=config \
      --namespace=external-istiod \
      --service-account=istiod \
      --create-service-account=false | \
      kubectl apply -f - --context="${CTX_EXTERNAL_CLUSTER}"
   {{< /text >}}

   {{< tip >}}
   `kind` で実行している場合は、`--server https://<api-server-node-ip>:6443`
   を `istioctl create-remote-secret` コマンドに渡す必要があります。
   ここで、`<api-server-node-ip>` は API サーバーを実行するノードの IP アドレスです。
   {{< /tip >}}

1. 外部クラスターの `external-istiod` 名前空間にコントロールプレーンをインストールするために Istio 設定を作成します。
   istiod がローカルでインストールされた `istio` ConfigMap を使用するように構成されており、
   `SHARED_MESH_CONFIG` 環境変数が `istio` に設定されていることに注意してください。
   これは、istiod がメッシュ管理者が設定クラスターの ConfigMap で設定した値とメッシュオペレーターがローカル ConfigMap で設定した値をマージすることを指示

   {{< text syntax=bash snip_id=get_external_istiod_iop >}}
   $ cat <<EOF > external-istiod.yaml
   apiVersion: install.istio.io/v1alpha1
   kind: IstioOperator
   metadata:
   namespace: external-istiod
   spec:
   profile: empty
   meshConfig:
   rootNamespace: external-istiod
   defaultConfig:
   discoveryAddress: $EXTERNAL_ISTIOD_ADDR:15012
   proxyMetadata:
   XDS_ROOT_CA: /etc/ssl/certs/ca-certificates.crt
   CA_ROOT_CA: /etc/ssl/certs/ca-certificates.crt
   components:
   pilot:
   enabled: true
   k8s:
   overlays: - kind: Deployment
   name: istiod
   patches: - path: spec.template.spec.volumes[100]
   value: |-
   name: config-volume
   configMap:
   name: istio - path: spec.template.spec.volumes[100]
   value: |-
   name: inject-volume
   configMap:
   name: istio-sidecar-injector - path: spec.template.spec.containers[0].volumeMounts[100]
   value: |-
   name: config-volume
   mountPath: /etc/istio/config - path: spec.template.spec.containers[0].volumeMounts[100]
   value: |-
   name: inject-volume
   mountPath: /var/lib/istio/inject
   env: - name: INJECTION_WEBHOOK_CONFIG_NAME
   value: "" - name: VALIDATION_WEBHOOK_CONFIG_NAME
   value: "" - name: EXTERNAL_ISTIOD
   value: "true" - name: LOCAL_CLUSTER_SECRET_WATCHER
   value: "true" - name: CLUSTER_ID
   value: ${REMOTE_CLUSTER_NAME} - name: SHARED_MESH_CONFIG
   value: istio
   values:
   global:
   caAddress: $EXTERNAL_ISTIOD_ADDR:15012
   istioNamespace: external-istiod
   operatorManageWebhooks: true
   configValidation: false
   meshID: mesh1
   EOF
   {{< /text >}}

1. `EXTERNAL_ISTIOD_ADDR` が IP アドレスではなく、正しい DNS ホスト名を使用している場合は、
   プロキシメタデータを削除し、Webhook 設定環境変数を更新します：

   {{< warning >}}
   本番環境では、これを行うことはお勧めしません。
   {{< /warning >}}

   {{< text bash >}}
   $ sed -i'.bk' \
    -e '/proxyMetadata:/,+2d' \
    -e '/INJECTION_WEBHOOK_CONFIG_NAME/{n;s/value: ""/value: istio-sidecar-injector-external-istiod/;}' \
    -e '/VALIDATION_WEBHOOK_CONFIG_NAME/{n;s/value: ""/value: istio-validator-external-istiod/;}' \
    external-istiod.yaml ; rm external-istiod.yaml.bk
   {{< /text >}}

1. 外部クラスター上で Istio 設定を適用します：

   {{< text bash >}}
   $ istioctl install -f external-istiod.yaml --context="${CTX_EXTERNAL_CLUSTER}"
   {{< /text >}}

1. 外部 Istiod が正常にデプロイされたことを確認します：

   {{< text bash >}}
   $ kubectl get po -n external-istiod --context="${CTX_EXTERNAL_CLUSTER}"
   NAME READY STATUS RESTARTS AGE
   istiod-779bd6fdcf-bd6rg 1/1 Running 0 70s
   {{< /text >}}

1. Istio `Gateway`、`VirtualService`、および `DestinationRule` 設定を作成し、
   Ingress Gateway から外部コントロールプレーンにトラフィックをルーティングします：

   {{< text syntax=bash snip_id=get_external_istiod_gateway_config >}}
   $ cat <<EOF > external-istiod-gw.yaml
   apiVersion: networking.istio.io/v1
   kind: Gateway
   metadata:
   name: external-istiod-gw
   namespace: external-istiod
   spec:
   selector:
   istio: ingressgateway
   servers: - port:
   number: 15012
   protocol: https
   name: https-XDS
   tls:
   mode: SIMPLE
   credentialName: $SSL_SECRET_NAME
   hosts: - $EXTERNAL_ISTIOD_ADDR - port:
   number: 15017
   protocol: https
   name: https-WEBHOOK
   tls:
   mode: SIMPLE
   credentialName: $SSL_SECRET_NAME
   hosts: - $EXTERNAL_ISTIOD_ADDR

   ***

   apiVersion: networking.istio.io/v1
   kind: VirtualService
   metadata:
   name: external-istiod-vs
   namespace: external-istiod
   spec:
   hosts: - $EXTERNAL_ISTIOD_ADDR
   gateways: - external-istiod-gw
   http: - match: - port: 15012
   route: - destination:
   host: istiod.external-istiod.svc.cluster.local
   port:
   number: 15012 - match: - port: 15017
   route: - destination:
   host: istiod.external-istiod.svc.cluster.local
   port:
   number: 443

   ***

   apiVersion: networking.istio.io/v1
   kind: DestinationRule
   metadata:
   name: external-istiod-dr
   namespace: external-istiod
   spec:
   host: istiod.external-istiod.svc.cluster.local
   trafficPolicy:
   portLevelSettings: - port:
   number: 15012
   tls:
   mode: SIMPLE
   connectionPool:
   http:
   h2UpgradePolicy: UPGRADE - port:
   number: 443
   tls:
   mode: SIMPLE
   EOF
   {{< /text >}}

1. `EXTERNAL_ISTIOD_ADDR` が IP アドレスではなく、正しい DNS ホスト名を使用している場合は、
   設定を変更します。
   `DestinationRule` を削除し、`Gateway` で TLS を終了せず、`VirtualService` で TLS ルーティングを使用します：

   {{< warning >}}
   本番環境では、これを行うことはお勧めしません。
   {{< /warning >}}

   {{< text bash >}}
   $ sed -i'.bk' \
    -e '55,$d' \
      -e 's/mode: SIMPLE/mode: PASSTHROUGH/' -e '/credentialName:/d' -e "s/${EXTERNAL*ISTIOD_ADDR}/\"*\"/" \
    -e 's/http:/tls:/' -e 's/https/tls/' -e '/route:/i\
    sniHosts:\

   - "\_"' \
      external-istiod-gw.yaml; rm external-istiod-gw.yaml.bk
     {{< /text >}}

1. 外部クラスター上で設定を適用します：

   {{< text bash >}}
   $ kubectl apply -f external-istiod-gw.yaml --context="${CTX_EXTERNAL_CLUSTER}"
   {{< /text >}}

### メッシュ管理者の手順 {#mesh-admin-steps}

Istio が起動して実行中になったので、メッシュ管理者はメッシュ内にサービスをデプロイし、
必要に応じて Gateway を含むように設定するだけです。

{{< tip >}}
デフォルトでは、リモートクラスター上では `istioctl` CLI コマンドが機能しませんが、
`istioctl` を簡単に設定して機能するようにすることができます。
詳細については、[Istioctl-proxy エコシステムプロジェクト](https://github.com/istio-ecosystem/istioctl-proxy-sample)を参照してください。
{{< /tip >}}

#### シンプルなアプリケーションをデプロイする {#deploy-a-sample-application}

1. リモートクラスター上で `sample` 名前空間を作成し、ラベル注入を有効にします：

   {{< text bash >}}
   $ kubectl create --context="${CTX_REMOTE_CLUSTER}" namespace sample
    $ kubectl label --context="${CTX_REMOTE_CLUSTER}" namespace sample istio-injection=enabled
   {{< /text >}}

1. サンプル `helloworld`（`v1`）と `curl` をデプロイします：

   {{< text bash >}}
   $ kubectl apply -f @samples/helloworld/helloworld.yaml@ -l service=helloworld -n sample --context="${CTX_REMOTE_CLUSTER}"
    $ kubectl apply -f @samples/helloworld/helloworld.yaml@ -l version=v1 -n sample --context="${CTX_REMOTE_CLUSTER}"
   $ kubectl apply -f @samples/curl/curl.yaml@ -n sample --context="${CTX_REMOTE_CLUSTER}"
   {{< /text >}}

1. 数秒後、Pod `helloworld` と `curl` が Sidecar 注入で実行されます：

   {{< text bash >}}
   $ kubectl get pod -n sample --context="${CTX_REMOTE_CLUSTER}"
   NAME READY STATUS RESTARTS AGE
   curl-64d7d56698-wqjnm 2/2 Running 0 9s
   helloworld-v1-5b75657f75-ncpc5 2/2 Running 0 10s
   {{< /text >}}

1. Pod `curl` から Pod `helloworld` サービスにリクエストを送信します：

   {{< text bash >}}
   $ kubectl exec --context="${CTX_REMOTE_CLUSTER}" -n sample -c curl \
        "$(kubectl get pod --context="${CTX_REMOTE_CLUSTER}" -n sample -l app=curl -o jsonpath='{.items[0].metadata.name}')" \
    -- curl -sS helloworld.sample:5000/hello
   Hello version: v1, instance: helloworld-v1-776f57d5f6-s7zfc
   {{< /text >}}

#### Gateway の有効化 {#enable-gateways}

{{< tip >}}
{{< boilerplate gateway-api-future >}}

Gateway API を使用している場合は、Gateway コンポーネントをインストールする必要はありません。
[Ingress Gateway の設定とテスト](#configure-and-test-an-ingress-gateway)に直接進むことができます。
{{< /tip >}}

リモートクラスター上で Ingress Gateway を有効にします：

{{< tabset category-name="ingress-gateway-install-type" >}}

{{< tab name="IstioOperator" category-value="iop" >}}

{{< text bash >}}
$ cat <<EOF > istio-ingressgateway.yaml
apiVersion: install.istio.io/v1alpha1
kind: IstioOperator
metadata:
name: ingress-install
spec:
profile: empty
components:
ingressGateways: - namespace: external-istiod
name: istio-ingressgateway
enabled: true
values:
gateways:
istio-ingressgateway:
injectionTemplate: gateway
EOF
$ istioctl install -f istio-ingressgateway.yaml --set values.global.istioNamespace=external-istiod --context="${CTX_REMOTE_CLUSTER}"
{{< /text >}}

{{< /tab >}}

{{< tab name="Helm" category-value="helm" >}}

{{< text bash >}}
$ helm install istio-ingressgateway istio/gateway -n external-istiod --kube-context="${CTX_REMOTE_CLUSTER}"
{{< /text >}}

Gateway のインストールについての詳細なドキュメントは、[Gateway のインストール](/ja/docs/setup/additional-setup/gateway/)を参照してください。

{{< /tab >}}
{{< /tabset >}}

リモートクラスター上で Egress Gateway またはその他の Gateway（オプション）を有効にします：

{{< tabset category-name="egress-gateway-install-type" >}}

{{< tab name="IstioOperator" category-value="iop" >}}

{{< text bash >}}
$ cat <<EOF > istio-egressgateway.yaml
apiVersion: install.istio.io/v1alpha1
kind: IstioOperator
metadata:
name: egress-install
spec:
profile: empty
components:
egressGateways: - namespace: external-istiod
name: istio-egressgateway
enabled: true
values:
gateways:
istio-egressgateway:
injectionTemplate: gateway
EOF
$ istioctl install -f istio-egressgateway.yaml --set values.global.istioNamespace=external-istiod --context="${CTX_REMOTE_CLUSTER}"
{{< /text >}}

{{< /tab >}}

{{< tab name="Helm" category-value="helm" >}}

{{< text bash >}}
$ helm install istio-egressgateway istio/gateway -n external-istiod --kube-context="${CTX_REMOTE_CLUSTER}" --set service.type=ClusterIP
{{< /text >}}

Gateway のインストールについての詳細なドキュメントは、[Gateway のインストール](/ja/docs/setup/additional-setup/gateway/)を参照してください。

{{< /tab >}}
{{< /tabset >}}

#### Ingress Gateway の設定とテスト {#configure-and-test-an-ingress-gateway}

{{< tip >}}
{{< boilerplate gateway-api-choose >}}
{{< /tip >}}

1. クラスターが Gateway の設定準備ができていることを確認します：

{{< tabset category-name="config-api" >}}

{{< tab name="Istio APIs" category-value="istio-apis" >}}

Istio Ingress Gateway が実行中であることを確認します：

{{< text bash >}}
$ kubectl get pod -l app=istio-ingressgateway -n external-istiod --context="${CTX_REMOTE_CLUSTER}"
NAME READY STATUS RESTARTS AGE
istio-ingressgateway-7bcd5c6bbd-kmtl4 1/1 Running 0 8m4s
{{< /text >}}

{{< /tab >}}

{{< tab name="Gateway API" category-value="gateway-api" >}}

デフォルトでは、Kubernetes Gateway API CRD はほとんどの Kubernetes クラスターにインストールされていません。
したがって、Gateway API を使用する前に、それらをインストールすることを確認してください：

{{< text syntax=bash snip_id=install_crds >}}
$ kubectl get crd gateways.gateway.networking.k8s.io --context="${CTX_REMOTE_CLUSTER}" || \
  { kubectl kustomize "github.com/kubernetes-sigs/gateway-api/config/crd?ref={{< k8s_gateway_api_version >}}" | kubectl apply -f - --context="${CTX_REMOTE_CLUSTER}"; }
{{< /text >}}

{{< /tab >}}

{{< /tabset >}}

2. Ingress Gateway 上で `helloworld` アプリケーションを公開します：

{{< tabset category-name="config-api" >}}

{{< tab name="Istio APIs" category-value="istio-apis" >}}

{{< text bash >}}
$ kubectl apply -f @samples/helloworld/helloworld-gateway.yaml@ -n sample --context="${CTX_REMOTE_CLUSTER}"
{{< /text >}}

{{< /tab >}}

{{< tab name="Gateway API" category-value="gateway-api" >}}

{{< text bash >}}
$ kubectl apply -f @samples/helloworld/gateway-api/helloworld-gateway.yaml@ -n sample --context="${CTX_REMOTE_CLUSTER}"
{{< /text >}}

{{< /tab >}}

{{< /tabset >}}

3. `GATEWAY_URL` 環境変数を設定します（詳細については、[Ingress の IP とポートを決定する](/ja/docs/tasks/traffic-management/ingress/ingress-control/#determining-the-ingress-ip-and-ports)を参照してください）：

{{< tabset category-name="config-api" >}}

{{< tab name="Istio APIs" category-value="istio-apis" >}}

{{< text bash >}}
$ export INGRESS_HOST=$(kubectl -n external-istiod --context="${CTX_REMOTE_CLUSTER}" get service istio-ingressgateway -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
$ export INGRESS_PORT=$(kubectl -n external-istiod --context="${CTX_REMOTE_CLUSTER}" get service istio-ingressgateway -o jsonpath='{.spec.ports[?(@.name=="http2")].port}')
$ export GATEWAY_URL=$INGRESS_HOST:$INGRESS_PORT
{{< /text >}}

{{< /tab >}}

{{< tab name="Gateway API" category-value="gateway-api" >}}

{{< text bash >}}
$ kubectl -n sample --context="${CTX_REMOTE_CLUSTER}" wait --for=condition=programmed gtw helloworld-gateway
$ export INGRESS_HOST=$(kubectl -n sample --context="${CTX_REMOTE_CLUSTER}" get gtw helloworld-gateway -o jsonpath='{.status.addresses[0].value}')
$ export GATEWAY_URL=$INGRESS_HOST:80
{{< /text >}}

{{< /tab >}}

{{< /tabset >}}

4. Ingress Gateway から `helloworld` アプリケーションにアクセスできることを確認します：

   {{< text bash >}}
   $ curl -s "http://${GATEWAY_URL}/hello"
   Hello version: v1, instance: helloworld-v1-776f57d5f6-s7zfc
   {{< /text >}}

## クラスターをメッシュに追加する（オプション） {#adding-clusters}

このセクションでは、別のリモートクラスターを追加して、既存の外部コントロールプレーンメッシュを多クラスターに拡張する方法を説明します。
これにより、サービスを簡単に配布し、[位置情報対応ルーティングとフェイルオーバー](/ja/docs/tasks/traffic-management/locality-load-balancing/)を使用して、アプリケーションの高可用性をサポートできます。

{{< image width="75%"
    link="external-multicluster.svg"
    caption="多リモートクラスターの外部コントロールプレーン"
    >}}

最初のリモートクラスターとは異なり、同じ外部コントロールプレーンに追加された 2 番目以降のクラスターは、メッシュ設定のみを提供しません。
[メインリモート](/ja/docs/setup/install/multicluster/primary-remote_multi-network/) Istio 多クラスター設定のリモートクラスターと同様です。

続行するには、Kubernetes クラスターをメッシュの 2 番目のリモートクラスターとして使用する必要があります。
以下の環境変数をクラスターのコンテキスト名とクラスター名に設定します：

{{< text syntax=bash snip_id=none >}}
$ export CTX_SECOND_CLUSTER=<2 番目のリモートクラスターのコンテキスト>
$ export SECOND_CLUSTER_NAME=<2 番目のリモートクラスターの名前>
{{< /text >}}

### 新しいクラスターを登録する {#register-the-new-cluster}

1. リモートクラスター上で Istio をインストールする設定を作成します。これは、ローカルでデプロイされた注入器ではなく、外部コントロールプレーン注入器を使用する注入 Webhook をインストールします：

   {{< text syntax=bash snip_id=get_second_remote_cluster_iop >}}
   $ cat <<EOF > second-remote-cluster.yaml
   apiVersion: install.istio.io/v1alpha1
   kind: IstioOperator
   metadata:
   namespace: external-istiod
   spec:
   profile: remote
   values:
   global:
   istioNamespace: external-istiod
   istiodRemote:
   injectionURL: https://${EXTERNAL_ISTIOD_ADDR}:15017/inject/cluster/${SECOND_CLUSTER_NAME}/net/network2
   EOF
   {{< /text >}}

1. `EXTERNAL_ISTIOD_ADDR` が IP アドレスではなく、正しい DNS ホスト名を使用している場合は、
   検出アドレスとパスを指定するように設定を変更します：

   {{< warning >}}
   本番環境では、これを行うことはお勧めしません。
   {{< /warning >}}

   {{< text bash >}}
   $ sed -i'.bk' \
    -e "s|injectionURL: https://${EXTERNAL_ISTIOD_ADDR}:15017|injectionPath: |" \
    -e "/istioNamespace:/a\\
   remotePilotAddress: ${EXTERNAL_ISTIOD_ADDR}" \
    second-remote-cluster.yaml; rm second-remote-cluster.yaml.bk
   {{< /text >}}

1. リモートクラスター上でシステム名前空間を作成し、注釈を追加します：

   {{< text bash >}}
   $ kubectl create namespace external-istiod --context="${CTX_SECOND_CLUSTER}"
    $ kubectl annotate namespace external-istiod "topology.istio.io/controlPlaneClusters=${REMOTE_CLUSTER_NAME}" --context="${CTX_SECOND_CLUSTER}"
   {{< /text >}}

   `topology.istio.io/controlPlaneClusters` 注解は、このリモートクラスターを管理するべき外部コントロールプレーンのクラスター ID を指定します。
   注意これは最初のリモート（設定）クラスターの名前であり、外部クラスターにインストールするときに外部コントロールプレーンのクラスター ID を設定するために使用されました。

1. リモートクラスター上で設定をインストールします：

   {{< text bash >}}
   $ istioctl install -f second-remote-cluster.yaml --context="${CTX_SECOND_CLUSTER}"
   {{< /text >}}

1. リモートクラスターの注入 Webhook 設定がインストールされていることを確認します：

   {{< text bash >}}
   $ kubectl get mutatingwebhookconfiguration --context="${CTX_SECOND_CLUSTER}"
   NAME WEBHOOKS AGE
   istio-sidecar-injector-external-istiod 4 4m13s
   {{< /text >}}

1. 資格情報を使用して Secret を作成し、外部コントロールプレーンが 2 番目のリモートクラスター上のエンドポイントにアクセスできるようにし、それをインストールします：

   {{< text bash >}}
   $ istioctl create-remote-secret \
    --context="${CTX_SECOND_CLUSTER}" \
      --name="${SECOND_CLUSTER_NAME}" \
    --type=remote \
    --namespace=external-istiod \
    --create-service-account=false | \
    kubectl apply -f - --context="${CTX_EXTERNAL_CLUSTER}"
   {{< /text >}}

   注意、メッシュの最初のリモートクラスターとは異なり、これも設定クラスターとして使用され、この時点で `--type` パラメーターは `remote` に設定されています。

### 東西向き Gateway を設定する {#setup-east-west-gateways}

1. 2 つのリモートクラスター上で東西向き Gateway をデプロイします：

   {{< text bash >}}
   $ @samples/multicluster/gen-eastwest-gateway.sh@ \
    --network network1 > eastwest-gateway-1.yaml
   $ istioctl manifest generate -f eastwest-gateway-1.yaml \
    --set values.global.istioNamespace=external-istiod | \
    kubectl apply --context="${CTX_REMOTE_CLUSTER}" -f -
   {{< /text >}}

   {{< text bash >}}
   $ @samples/multicluster/gen-eastwest-gateway.sh@ \
    --network network2 > eastwest-gateway-2.yaml
   $ istioctl manifest generate -f eastwest-gateway-2.yaml \
    --set values.global.istioNamespace=external-istiod | \
    kubectl apply --context="${CTX_SECOND_CLUSTER}" -f -
   {{< /text >}}

1. 東西向き Gateway が外部 IP アドレスを割り当てられるのを待ちます：

   {{< text bash >}}
   $ kubectl --context="${CTX_REMOTE_CLUSTER}" get svc istio-eastwestgateway -n external-istiod
   NAME TYPE CLUSTER-IP EXTERNAL-IP PORT(S) AGE
   istio-eastwestgateway LoadBalancer 10.0.12.121 34.122.91.98 ... 51s
   {{< /text >}}

   {{< text bash >}}
   $ kubectl --context="${CTX_SECOND_CLUSTER}" get svc istio-eastwestgateway -n external-istiod
   NAME TYPE CLUSTER-IP EXTERNAL-IP PORT(S) AGE
   istio-eastwestgateway LoadBalancer 10.0.12.121 34.122.91.99 ... 51s
   {{< /text >}}

1. 東西向き Gateway を使用してサービスを公開します：

   {{< text bash >}}
   $ kubectl --context="${CTX_REMOTE_CLUSTER}" apply -n external-istiod -f \
    @samples/multicluster/expose-services.yaml@
   {{< /text >}}

### インストールを検証する {#validate-the-installation}

1. リモートクラスター上で `sample` 名前空間を作成し、ラベル注入を有効にします：

   {{< text bash >}}
   $ kubectl create --context="${CTX_SECOND_CLUSTER}" namespace sample
    $ kubectl label --context="${CTX_SECOND_CLUSTER}" namespace sample istio-injection=enabled
   {{< /text >}}

1. `helloworld`（`v2` バージョン）と `curl` のサンプルをデプロイします：

   {{< text bash >}}
   $ kubectl apply -f @samples/helloworld/helloworld.yaml@ -l service=helloworld -n sample --context="${CTX_SECOND_CLUSTER}"
    $ kubectl apply -f @samples/helloworld/helloworld.yaml@ -l version=v2 -n sample --context="${CTX_SECOND_CLUSTER}"
   $ kubectl apply -f @samples/curl/curl.yaml@ -n sample --context="${CTX_SECOND_CLUSTER}"
   {{< /text >}}

1. 数秒後、Pod `helloworld` と `curl` が Sidecar 注入で実行されます：

   {{< text bash >}}
   $ kubectl get pod -n sample --context="${CTX_SECOND_CLUSTER}"
   NAME READY STATUS RESTARTS AGE
   curl-557747455f-wtdbr 2/2 Running 0 9s
   helloworld-v2-54df5f84b-9hxgw 2/2 Running 0 10s
   {{< /text >}}

   1. Pod `curl` から `helloworld` サービスにリクエストを送信します：

   {{< text bash >}}
   $ kubectl exec --context="${CTX_SECOND_CLUSTER}" -n sample -c curl \
        "$(kubectl get pod --context="${CTX_SECOND_CLUSTER}" -n sample -l app=curl -o jsonpath='{.items[0].metadata.name}')" \
    -- curl -sS helloworld.sample:5000/hello
   Hello version: v2, instance: helloworld-v2-54df5f84b-9hxgw
   {{< /text >}}

1. Ingress Gateway から `helloworld` アプリケーションに複数回アクセスすると、現在 `v1` と `v2` のバージョンが呼び出されます：

   {{< text bash >}}
   $ for i in {1..10}; do curl -s "http://${GATEWAY_URL}/hello"; done
   Hello version: v1, instance: helloworld-v1-776f57d5f6-s7zfc
   Hello version: v2, instance: helloworld-v2-54df5f84b-9hxgw
   Hello version: v1, instance: helloworld-v1-776f57d5f6-s7zfc
   Hello version: v2, instance: helloworld-v2-54df5f84b-9hxgw
   ...
   {{< /text >}}

## 環境のクリーンアップ {#clean-up}

外部コントロールプレーンクラスターをクリーンアップします：

{{< text bash >}}
$ kubectl delete -f external-istiod-gw.yaml --context="${CTX_EXTERNAL_CLUSTER}"
$ istioctl uninstall -y --purge -f external-istiod.yaml --context="${CTX_EXTERNAL_CLUSTER}"
$ kubectl delete ns istio-system external-istiod --context="${CTX_EXTERNAL_CLUSTER}"
$ rm controlplane-gateway.yaml external-istiod.yaml external-istiod-gw.yaml
{{< /text >}}

リモート設定クラスターをクリーンアップします：

{{< text bash >}}
$ kubectl delete ns sample --context="${CTX_REMOTE_CLUSTER}"
$ istioctl uninstall -y --purge -f remote-config-cluster.yaml --set values.defaultRevision=default --context="${CTX_REMOTE_CLUSTER}"
$ kubectl delete ns external-istiod --context="${CTX_REMOTE_CLUSTER}"
$ rm remote-config-cluster.yaml istio-ingressgateway.yaml
$ rm istio-egressgateway.yaml eastwest-gateway-1.yaml || true
{{< /text >}}

オプションの 2 番目のリモートクラスターがインストールされている場合は、それをクリーンアップします：

{{< text bash >}}
$ kubectl delete ns sample --context="${CTX_SECOND_CLUSTER}"
$ istioctl uninstall -y --purge -f second-remote-cluster.yaml --context="${CTX_SECOND_CLUSTER}"
$ kubectl delete ns external-istiod --context="${CTX_SECOND_CLUSTER}"
$ rm second-remote-cluster.yaml eastwest-gateway-2.yaml
{{< /text >}}
