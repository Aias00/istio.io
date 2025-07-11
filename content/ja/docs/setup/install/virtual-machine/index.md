---
title: 仮想マシンインストール
description: Istio をデプロイし、仮想マシン上で動作するワークロードをメッシュに参加させます。
weight: 60
keywords:
  - kubernetes
  - virtual-machine
  - gateways
  - vms
owner: istio/wg-environments-maintainers
test: yes
---

このガイドに従って Istio をデプロイし、仮想マシンをメッシュに参加させてください。

## 前提条件 {#prerequisites}

1. [Istio リリースをダウンロード](/zh/docs/setup/additional-setup/download-istio-release/)
1. 必要な[プラットフォームセットアップ](/zh/docs/setup/platform-setup/)を実行
1. [Pod と Service の要件](/zh/docs/ops/deployment/application-requirements/)を確認
1. 仮想マシンは、ターゲットメッシュのイングレスゲートウェイに IP 到達可能である必要があります。より高いパフォーマンス要件がある場合は、メッシュ内の各 Pod へのレイヤ 3 ネットワーク接続も可能です。
1. [仮想マシンアーキテクチャ](/zh/docs/ops/deployment/vm-architecture/)を読んで、Istio 仮想マシン統合のアーキテクチャ概要を理解してください。

## ガイド用環境の準備 {#prepare-the-guide-environment}

1. 仮想マシンを作成
1. クラスタのマシン上で環境変数 `VM_APP`、`WORK_DIR`、`VM_NAMESPACE`、`SERVICE_ACCOUNT` を設定します
   （例：`WORK_DIR="${HOME}/vmintegration"`）：

   {{< tabset category-name="network-mode" >}}

   {{< tab name="単一ネットワーク" category-value="single" >}}

   {{< text bash >}}
   $ VM_APP="<この仮想マシンで動作するアプリ名>"
   $ VM_NAMESPACE="<サービスが属するネームスペース>"
   $ WORK_DIR="<証明書作業ディレクトリ>"
   $ SERVICE_ACCOUNT="<この仮想マシン用の Kubernetes サービスアカウント名>"
   $ CLUSTER_NETWORK=""
   $ VM_NETWORK=""
   $ CLUSTER="Kubernetes"
   {{< /text >}}

   {{< /tab >}}

   {{< tab name="複数ネットワーク" category-value="multiple" >}}

   {{< text bash >}}
   $ VM_APP="<この仮想マシンで動作するアプリ名>"
   $ VM_NAMESPACE="<サービスが属するネームスペース>"
   $ WORK_DIR="<証明書作業ディレクトリ>"
   $ SERVICE_ACCOUNT="<この仮想マシン用の Kubernetes サービスアカウント名>"
   $ # 必要に応じてマルチクラスタ/マルチネットワークのパラメータをカスタマイズ
   $ CLUSTER_NETWORK="kube-network"
   $ VM_NETWORK="vm-network"
   $ CLUSTER="cluster1"
   {{< /text >}}

   {{< /tab >}}

   {{< /tabset >}}

1. 作業ディレクトリを作成：

   {{< text syntax=bash snip_id=setup_wd >}}
   $ mkdir -p "${WORK_DIR}"
   {{< /text >}}

## Istio コントロールプレーンのインストール {#install-control-plane}

クラスタにすでに Istio コントロールプレーンがある場合は、インストール手順をスキップできますが、仮想マシンからコントロールプレーンへの公開アクセスは必要です。

Istio をインストールし、コントロールプレーンへの外部アクセスを有効にして、仮想マシンがアクセスできるようにします。

1. Istio インストール用の `IstioOperator` を作成します。

   {{< text syntax="bash yaml" snip_id=setup_iop >}}
   $ cat <<EOF > ./vm-cluster.yaml
   apiVersion: install.istio.io/v1alpha1
   kind: IstioOperator
   metadata:
   name: istio
   spec:
   values:
   global:
   meshID: mesh1
   multiCluster:
   clusterName: "${CLUSTER}"
          network: "${CLUSTER_NETWORK}"
   EOF
   {{< /text >}}

1. Istio をインストールします。

   {{< tabset category-name="registration-mode" >}}

   {{< tab name="デフォルト" category-value="default" >}}

   {{< text bash >}}
   $ istioctl install -f vm-cluster.yaml
   {{< /text >}}

   {{< /tab >}}

   {{< tab name="WorkloadEntry 自動作成" category-value="autoreg" >}}

   {{< boilerplate experimental >}}

   {{< text bash >}}
   $ istioctl install -f vm-cluster.yaml --set values.pilot.env.PILOT_ENABLE_WORKLOAD_ENTRY_AUTOREGISTRATION=true --set values.pilot.env.PILOT_ENABLE_WORKLOAD_ENTRY_HEALTHCHECKS=true
   {{< /text >}}

   {{< /tab >}}

   {{< /tabset >}}

1. イーストウエストゲートウェイをデプロイ：

   {{< warning >}}
   コントロールプレーンがリビジョン付きでインストールされている場合は、`gen-eastwest-gateway.sh` コマンドに `--revision rev` パラメータを追加してください。
   {{< /warning >}}

   {{< tabset category-name="network-mode" >}}

   {{< tab name="単一ネットワーク" category-value="single" >}}

   {{< text syntax=bash snip_id=install_eastwest >}}
   $ @samples/multicluster/gen-eastwest-gateway.sh@ --single-cluster | istioctl install -y -f -
   {{< /text >}}

   {{< /tab >}}

   {{< tab name="複数ネットワーク" category-value="multiple" >}}

   {{< text bash >}}
   $ @samples/multicluster/gen-eastwest-gateway.sh@ \
   --network "${CLUSTER_NETWORK}" | \
   istioctl install -y -f -
   {{< /text >}}

   {{< /tab >}}

   {{< /tabset >}}

1. イーストウエストゲートウェイを使ってクラスタ内サービスを公開：

   {{< tabset category-name="network-mode" >}}

   {{< tab name="単一ネットワーク" category-value="single" >}}

   コントロールプレーンを公開：

   {{< text syntax=bash snip_id=expose_istio >}}
   $ kubectl apply -f @samples/multicluster/expose-istiod.yaml@
   {{< /text >}}

   {{< /tab >}}

   {{< tab name="複数ネットワーク" category-value="multiple" >}}

   コントロールプレーンを公開：

   {{< text bash >}}
   $ kubectl apply -f @samples/multicluster/expose-istiod.yaml@
   {{< /text >}}

   クラスタサービスを公開：

   {{< text bash >}}
   $ kubectl apply -n istio-system -f @samples/multicluster/expose-services.yaml@
   {{< /text >}}

   istio-system ネームスペースに定義したクラスタネットワークのラベルを付与してください：

   {{< text bash >}}
   $ kubectl label namespace istio-system topology.istio.io/network="${CLUSTER_NETWORK}"
   {{< /text >}}

   {{< /tab >}}

   {{< /tabset >}}

## 仮想マシン用ネームスペースの設定 {#configure-the-virtual-machine-namespace}

1. 仮想マシン用のネームスペースを作成：

   {{< text syntax=bash snip_id=install_namespace >}}
   $ kubectl create namespace "${VM_NAMESPACE}"
   {{< /text >}}

1. 仮想マシン用の ServiceAccount を作成：

   {{< text syntax=bash snip_id=install_sa >}}
   $ kubectl create serviceaccount "${SERVICE_ACCOUNT}" -n "${VM_NAMESPACE}"
   {{< /text >}}

## 仮想マシンに転送するファイルの作成 {#create-files-to-transfer-to-the-virtual-machine}

{{< tabset category-name="registration-mode" >}}

{{< tab name="デフォルト" category-value="default" >}}

まず、仮想マシン用の `WorkloadGroup` テンプレートを作成します：

{{< text bash >}}
$ cat <<EOF > workloadgroup.yaml
apiVersion: networking.istio.io/v1
kind: WorkloadGroup
metadata:
name: "${VM_APP}"
  namespace: "${VM_NAMESPACE}"
spec:
metadata:
labels:
app: "${VM_APP}"
  template:
    serviceAccount: "${SERVICE_ACCOUNT}"
network: "${VM_NETWORK}"
EOF
{{< /text >}}

{{< /tab >}}

{{< tab name="WorkloadEntry 自動作成" category-value="autoreg" >}}

まず、仮想マシン用の `WorkloadGroup` テンプレートを作成します：

{{< boilerplate experimental >}}

{{< text syntax=bash snip_id=create_wg >}}
$ cat <<EOF > workloadgroup.yaml
apiVersion: networking.istio.io/v1
kind: WorkloadGroup
metadata:
name: "${VM_APP}"
  namespace: "${VM_NAMESPACE}"
spec:
metadata:
labels:
app: "${VM_APP}"
  template:
    serviceAccount: "${SERVICE_ACCOUNT}"
network: "${VM_NETWORK}"
EOF
{{< /text >}}

次に、`WorkLoadGroup` をクラスタに適用します：

{{< text syntax=bash snip_id=apply_wg >}}
$ kubectl --namespace "${VM_NAMESPACE}" apply -f workloadgroup.yaml
{{< /text >}}

WorkloadEntry の自動作成機能を使うと、アプリケーションのヘルスチェックも可能です。
[Kubernetes Readiness Probes](https://kubernetes.io/zh-cn/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/) と同じ動作・API です。

例えば、アプリケーションの `/ready` エンドポイントでプローブを設定する場合：

{{< text bash >}}
$ cat <<EOF > workloadgroup.yaml
apiVersion: networking.istio.io/v1
kind: WorkloadGroup
metadata:
name: "${VM_APP}"
  namespace: "${VM_NAMESPACE}"
spec:
metadata:
labels:
app: "${VM_APP}"
  template:
    serviceAccount: "${SERVICE_ACCOUNT}"
network: "${NETWORK}"
probe:
periodSeconds: 5
initialDelaySeconds: 1
httpGet:
port: 8080
path: /ready
EOF
{{< /text >}}

この設定により、プローブが成功するまで自動生成された `WorkloadEntry` は "Ready" としてマークされません。

{{< /tab >}}

{{< /tabset >}}

{{< warning >}}
`istioctl x workload entry` で `istio-token` を生成する前に、
[ドキュメント](/zh/docs/ops/best-practices/security/#configure-third-party-service-account-tokens)に従って、クラスタでサードパーティサービスアカウントトークンが使われているか確認してください。もし使われていない場合は、Istio インストールコマンドに `--set values.global.jwtPolicy=first-party-jwt` を追加してください。
{{< /warning >}}

次に、`istioctl x workload entry` コマンドを使って以下を生成します：

- `cluster.env`：ネームスペース、サービスアカウント、ネットワーク CIDR、インバウンドポート（オプション）などのメタデータを含みます。
- `istio-token`：CA から証明書を取得するための Kubernetes トークン。
- `mesh.yaml`：`ProxyConfig` を提供し、`discoveryAddress`、ヘルスチェック、認証操作などを設定します。
- `root-cert.pem`：認証用のルート証明書。
- `hosts`：`/etc/hosts` の補助ファイルで、プロキシが Istiod から xDS を取得する際に使用します。

{{< idea >}}
より複雑なオプションとして、仮想マシンで外部 DNS サーバを参照するように DNS を設定する方法もあります。
このオプションは本ガイドの範囲外です。
{{< /idea >}}

{{< tabset category-name="registration-mode" >}}

{{< tab name="デフォルト" category-value="default" >}}

{{< text bash >}}
$ istioctl x workload entry configure -f workloadgroup.yaml -o "${WORK_DIR}" --clusterID "${CLUSTER}"
{{< /text >}}

{{< /tab >}}

{{< tab name="WorkloadEntry 自動作成" category-value="autoreg" >}}

{{< boilerplate experimental >}}

{{< text syntax=bash snip_id=configure_wg >}}
$ istioctl x workload entry configure -f workloadgroup.yaml -o "${WORK_DIR}" --clusterID "${CLUSTER}" --autoregister
{{< /text >}}

{{< /tab >}}

{{< /tabset >}}

## 仮想マシンの設定 {#configure-the-virtual-machine}

Istio メッシュに追加する仮想マシン上で、以下のコマンドを実行します：

1. "${WORK_DIR}" から仮想マシンへファイルを安全にアップロードします。安全な転送方法は、セキュリティポリシーに従ってください。本ガイドでは、すべての必須ファイルを仮想マシンの "${HOME}" ディレクトリにアップロードする例を示します。

1. ルート証明書を `/etc/certs` ディレクトリにインストール：

   {{< text bash >}}
   $ sudo mkdir -p /etc/certs
   $ sudo cp "${HOME}"/root-cert.pem /etc/certs/root-cert.pem
   {{< /text >}}

1. トークンを `/var/run/secrets/tokens` ディレクトリにインストール：

   {{< text bash >}}
   $ sudo mkdir -p /var/run/secrets/tokens
   $ sudo cp "${HOME}"/istio-token /var/run/secrets/tokens/istio-token
   {{< /text >}}

1. Istio 仮想マシン統合ランタイムを含むパッケージをインストール：

   {{< tabset category-name="vm-os" >}}

   {{< tab name="Debian" category-value="debian" >}}

   {{< text bash >}}
   $ curl -LO https://storage.googleapis.com/istio-release/releases/{{< istio_full_version >}}/deb/istio-sidecar.deb
   $ sudo dpkg -i istio-sidecar.deb
   {{< /text >}}

   {{< /tab >}}

   {{< tab name="CentOS" category-value="centos" >}}

   注意：現在サポートされているのは CentOS 8 のみです。

   {{< text bash >}}
   $ curl -LO https://storage.googleapis.com/istio-release/releases/{{< istio_full_version >}}/rpm/istio-sidecar.rpm
   $ sudo rpm -i istio-sidecar.rpm
   {{< /text >}}

   {{< /tab >}}

   {{< /tabset >}}

1. `cluster.env` を `/var/lib/istio/envoy/` ディレクトリにインストール：

   {{< text bash >}}
   $ sudo cp "${HOME}"/cluster.env /var/lib/istio/envoy/cluster.env
   {{< /text >}}

1. メッシュ設定ファイル [Mesh Config](/zh/docs/reference/config/istio.mesh.v1alpha1/#MeshConfig) を `/etc/istio/config/mesh` ディレクトリにインストール：

   {{< text bash >}}
   $ sudo cp "${HOME}"/mesh.yaml /etc/istio/config/mesh
   {{< /text >}}

1. istiod ホストを `/etc/hosts` に追加：

   {{< text bash >}}
   $ sudo sh -c 'cat $(eval echo ~$SUDO_USER)/hosts >> /etc/hosts'
   {{< /text >}}

1. `/etc/certs/` および `/var/lib/istio/envoy/` の所有権を Istio プロキシに変更：

   {{< text bash >}}
   $ sudo mkdir -p /etc/istio/proxy
   $ sudo chown -R istio-proxy /var/lib/istio /etc/certs /etc/istio/proxy /etc/istio/config /var/run/secrets /etc/certs/root-cert.pem
   {{< /text >}}

## 仮想マシンで Istio を起動 {#start-within-the-virtual-machine}

1. Istio プロキシを起動：

   {{< text bash >}}
   $ sudo systemctl start istio
   {{< /text >}}

## Istio が正常に動作しているかの確認 {#verify-works-successfully}

1. `/var/log/istio/istio.log` のログを確認し、以下のような内容が出力されていれば正常です：

   {{< text bash >}}
   $ 2020-08-21T01:32:17.748413Z info sds resource:default pushed key/cert pair to proxy
   $ 2020-08-21T01:32:20.270073Z info sds resource:ROOTCA new connection
   $ 2020-08-21T01:32:20.270142Z info sds Skipping waiting for gateway secret
   $ 2020-08-21T01:32:20.270279Z info cache adding watcher for file ./etc/certs/root-cert.pem
   $ 2020-08-21T01:32:20.270347Z info cache GenerateSecret from file ROOTCA
   $ 2020-08-21T01:32:20.270494Z info sds resource:ROOTCA pushed root cert to proxy
   $ 2020-08-21T01:32:20.270734Z info sds resource:default new connection
   $ 2020-08-21T01:32:20.270763Z info sds Skipping waiting for gateway secret
   $ 2020-08-21T01:32:20.695478Z info cache GenerateSecret default
   $ 2020-08-21T01:32:20.695595Z info sds resource:default pushed key/cert pair to proxy
   {{< /text >}}

1. Pod ベースのサービスをデプロイするためのネームスペースを作成：

   {{< text bash >}}
   $ kubectl create namespace sample
   $ kubectl label namespace sample istio-injection=enabled
   {{< /text >}}

1. `HelloWorld` サービスをデプロイ：

   {{< text bash >}}
   $ kubectl apply -f @samples/helloworld/helloworld.yaml@
   {{< /text >}}

1. 仮想マシンからサービスにリクエストを送信：

   {{< text bash >}}
   $ curl helloworld.sample.svc:5000/hello
   Hello version: v1, instance: helloworld-v1-578dd69f69-fxwwk
   {{< /text >}}

## 次のステップ {#next-step}

仮想マシンに関する詳細情報：

- [仮想マシンのデバッグ](/zh/docs/ops/diagnostic-tools/virtual-machines/)で仮想マシンの問題を解決
- [仮想マシン上の Bookinfo デプロイ](/zh/docs/examples/virtual-machines/)で仮想マシンのサンプルデプロイを確認

## アンインストール {#uninstall}

仮想マシンで Istio を停止：

{{< text bash >}}
$ sudo systemctl stop istio
{{< /text >}}

次に、Istio-sidecar のパッケージを削除：

{{< tabset category-name="vm-os" >}}

{{< tab name="Debian" category-value="debian" >}}

{{< text bash >}}
$ sudo dpkg -r istio-sidecar
$ dpkg -s istio-sidecar
{{< /text >}}

{{< /tab >}}

{{< tab name="CentOS" category-value="centos" >}}

{{< text bash >}}
$ sudo rpm -e istio-sidecar
{{< /text >}}

{{< /tab >}}

{{< /tabset >}}

Istio をアンインストールするには、次のコマンドを実行してください：

{{< text bash >}}
$ kubectl delete -f @samples/multicluster/expose-istiod.yaml@
$ istioctl uninstall -y --purge
{{< /text >}}

デフォルトでは、コントロールプレーンのネームスペース（例：`istio-system`）は削除されません。
不要な場合は、以下のコマンドで削除してください：

{{< text bash >}}
$ kubectl delete namespace istio-system
{{< /text >}}
