---
title: プラットフォーム固有の前提条件
description: Ambient モードの Istio をインストールする際のプラットフォーム固有の前提条件。
weight: 2
aliases:
  - /ja/docs/ops/ambient/install/platform-prerequisites
  - /ja/latest/docs/ops/ambient/install/platform-prerequisites
owner: istio/wg-environments-maintainers
test: no
---

本ドキュメントでは、Ambient モードの Istio をインストールする際の各種プラットフォームや環境固有の前提条件について説明します。

## プラットフォーム {#platform}

一部の Kubernetes 環境では、Istio をサポートするためにさまざまな設定オプションを構成する必要があります。

### Google Kubernetes Engine（GKE） {#google-kubernetes-engine-gke}

#### ネームスペースの制限 {#namespace-restrictions}

GKE では、[system-node-critical](https://kubernetes.io/ja/docs/tasks/administer-cluster/guaranteed-scheduling-critical-addon-pods/) `priorityClassName` を持つ Pod は、
[ResourceQuota](https://kubernetes.io/ja/docs/concepts/policy/resource-quotas/) が定義されたネームスペースにのみインストールできます。
デフォルトでは、GKE では `kube-system` のみが `node-critical` クラスの ResourceQuota を定義しています。
Istio CNI ノードエージェントと `ztunnel` の両方が `node-critical` クラスを必要とするため、GKE では次のいずれかの条件を満たす必要があります：

- `kube-system`（**`istio-system` ではありません**）にインストールする
- 手動で ResourceQuota を作成した別のネームスペース（例：`istio-system`）にインストールする。例：

{{< text syntax=yaml >}}
apiVersion: v1
kind: ResourceQuota
metadata:
name: gcp-critical-pods
namespace: istio-system
spec:
hard:
pods: 1000
scopeSelector:
matchExpressions: - operator: In
scopeName: PriorityClass
values: - system-node-critical
{{< /text >}}

#### プラットフォームプロファイル {#platform-profile}

GKE を使用する場合、CNI バイナリの非標準パスのため、インストールコマンドに正しい `platform` 値を追加し、Helm のオーバーライドが必要です。

{{< tabset category-name="install-method" >}}

{{< tab name="Helm" category-value="helm" >}}

    {{< text syntax=bash >}}
    $ helm install istio-cni istio/cni -n istio-system --set profile=ambient --set global.platform=gke --wait
    {{< /text >}}

{{< /tab >}}

{{< tab name="istioctl" category-value="istioctl" >}}

    {{< text syntax=bash >}}
    $ istioctl install --set profile=ambient --set values.global.platform=gke
    {{< /text >}}

{{< /tab >}}

{{< /tabset >}}

### Amazon Elastic Kubernetes Service（EKS） {#amazon-elastic-kubernetes-service-EKS}

EKS を使用している場合：

- Amazon の VPC CNI を使用している
- Pod ENI リレーを有効にしている
- **かつ** [SecurityGroupPolicy](https://aws.github.io/aws-eks-best-practices/networking/sgpp/#enforcing-mode-use-strict-mode-for-isolating-pod-and-node-traffic) を使って EKS Pod にセキュリティグループを割り当てている

[`POD_SECURITY_GROUP_ENFORCING_MODE` を明示的に `standard` に設定する必要があります](https://github.com/aws/amazon-vpc-cni-k8s/blob/master/README.md#pod_security_group_enforcing_mode-v1110)。
そうしないと Pod のヘルスチェックが失敗します。これは、Istio が kubelet のヘルスチェックを識別するためにリンクローカル SNAT アドレスを使用しており、
VPC CNI が Pod セキュリティグループの `strict` モードでリンクローカルパケットを誤ってルーティングするためです。
リンクローカルアドレスの CIDR 除外をセキュリティグループに追加しても効果はありません。
VPC CNI の Pod セキュリティグループモードは、リレー Pod ENI を経由してトラフィックをルーティングし、
セキュリティグループポリシーを適用するため、
[リンクローカルトラフィックはリンクをまたいでルーティングできません](https://datatracker.ietf.org/doc/html/rfc3927#section-2.6.2)。
このため、Pod セキュリティグループ機能はこれらのトラフィックにポリシーを適用できず、`strict` モードではパケットが破棄されます。

[VPC CNI コンポーネントには未解決の問題があります](https://github.com/aws/amazon-vpc-cni-k8s/issues/2797)。
Pod セキュリティグループを使用している場合、VPC CNI チームの現時点での推奨は `strict` モードを無効にするか、
Pod で kubelet ベースではなく exec ベースの Kubernetes プローブを使用することです。

Pod ENI リレーが有効かどうかは、次のコマンドで確認できます：

{{< text syntax=bash >}}
$ kubectl set env daemonset aws-node -n kube-system --list | grep ENABLE_POD_ENI
{{< /text >}}

クラスター内に Pod に割り当てられたセキュリティグループがあるかどうかは、次のコマンドで確認できます：

{{< text syntax=bash >}}
$ kubectl get securitygrouppolicies.vpcresources.k8s.aws
{{< /text >}}

`POD_SECURITY_GROUP_ENFORCING_MODE=standard` を設定し、影響を受ける Pod を再起動するには、次のコマンドを実行します：

{{< text syntax=bash >}}
$ kubectl set env daemonset aws-node -n kube-system POD_SECURITY_GROUP_ENFORCING_MODE=standard
{{< /text >}}

### k3d

[k3d](https://k3d.io/) とデフォルトの Flannel CNI を使用する場合、
CNI 設定やバイナリのパスが非標準のため、インストールコマンドに正しい `platform` 値を追加し、Helm のオーバーライドが必要です。

1.  Traefik を無効化したクラスターを作成し、Istio のイングレスゲートウェイと競合しないようにします：

    {{< text bash >}}
    $ k3d cluster create --api-port 6550 -p '9080:80@loadbalancer' -p '9443:443@loadbalancer' --agents 2 --k3s-arg '--disable=traefik@server:\*'
    {{< /text >}}

1.  Istio Chart をインストールする際に `global.platform=k3d` を指定します。例：

    {{< tabset category-name="install-method" >}}

    {{< tab name="Helm" category-value="helm" >}}

        {{< text syntax=bash >}}
        $ helm install istio-cni istio/cni -n istio-system --set profile=ambient --set global.platform=k3d --wait
        {{< /text >}}

    {{< /tab >}}

    {{< tab name="istioctl" category-value="istioctl" >}}

        {{< text syntax=bash >}}
        $ istioctl install --set profile=ambient --set values.global.platform=k3d
        {{< /text >}}

    {{< /tab >}}

    {{< /tabset >}}

### K3s

[K3s](https://k3s.io/) およびバンドルされている CNI のいずれかを使用する場合、
CNI 設定やバイナリのパスが非標準のため、インストールコマンドに正しい `platform` 値を追加し、Helm のオーバーライドが必要です。
K3s のデフォルトパスについては、Istio は `global.platform` の値に応じた組み込みオーバーライドを提供しています。

{{< tabset category-name="install-method" >}}

{{< tab name="Helm" category-value="helm" >}}

    {{< text syntax=bash >}}
    $ helm install istio-cni istio/cni -n istio-system --set profile=ambient --set global.platform=k3s --wait
    {{< /text >}}

{{< /tab >}}

{{< tab name="istioctl" category-value="istioctl" >}}

    {{< text syntax=bash >}}
    $ istioctl install --set profile=ambient --set values.global.platform=k3s
    {{< /text >}}

{{< /tab >}}

{{< /tabset >}}

ただし、K3s のドキュメントによると、これらのパスは K3s で上書きされる場合があります。
K3s でカスタムまたはバンドルされていない CNI を使用する場合は、
CNI の正しいパス（例：`/etc/cni/net.d`）を手動で指定する必要があります。
[詳細は K3s ドキュメントを参照](https://docs.k3s.io/ja/networking/basic-network-options#custom-cni)。例：

{{< tabset category-name="install-method" >}}

{{< tab name="Helm" category-value="helm" >}}

    {{< text syntax=bash >}}
    $ helm install istio-cni istio/cni -n istio-system --set profile=ambient --wait --set cniConfDir=/var/lib/rancher/k3s/agent/etc/cni/net.d --set cniBinDir=/var/lib/rancher/k3s/data/current/bin/
    {{< /text >}}

{{< /tab >}}

{{< tab name="istioctl" category-value="istioctl" >}}

    {{< text syntax=bash >}}
    $ istioctl install --set profile=ambient --set values.cni.cniConfDir=/var/lib/rancher/k3s/agent/etc/cni/net.d --set values.cni.cniBinDir=/var/lib/rancher/k3s/data/current/bin/
    {{< /text >}}

{{< /tab >}}

{{< /tabset >}}

### MicroK8s

[MicroK8s](https://microk8s.io/) で Istio をインストールする場合、
インストールコマンドに正しい `platform` 値を追加する必要があります。
MicroK8s は [CNI 設定やバイナリのパスが非標準](https://microk8s.io/docs/change-cidr) です。例：

{{< tabset category-name="install-method" >}}

{{< tab name="Helm" category-value="helm" >}}

    {{< text syntax=bash >}}
    $ helm install istio-cni istio/cni -n istio-system --set profile=ambient --set global.platform=microk8s --wait

    {{< /text >}}

{{< /tab >}}

{{< tab name="istioctl" category-value="istioctl" >}}

    {{< text syntax=bash >}}
    $ istioctl install --set profile=ambient --set values.global.platform=microk8s
    {{< /text >}}

{{< /tab >}}

{{< /tabset >}}

### minikube

[minikube](https://kubernetes.io/ja/docs/tasks/tools/install-minikube/) と [Docker ドライバー](https://minikube.sigs.k8s.io/docs/drivers/docker/) を使用する場合、
minikube（Docker 利用時）は非標準のコンテナバインドマウントパスを使用するため、インストールコマンドに正しい `platform` 値を追加する必要があります。例：

{{< tabset category-name="install-method" >}}

{{< tab name="Helm" category-value="helm" >}}

    {{< text syntax=bash >}}
    $ helm install istio-cni istio/cni -n istio-system --set profile=ambient --set global.platform=minikube --wait"
    {{< /text >}}

{{< /tab >}}

{{< tab name="istioctl" category-value="istioctl" >}}

    {{< text syntax=bash >}}
    $ istioctl install --set profile=ambient --set values.global.platform=minikube"
    {{< /text >}}

{{< /tab >}}

{{< /tabset >}}

### Red Hat OpenShift {#red-hat-openshift}

OpenShift では、`ztunnel` および `istio-cni` コンポーネントを `kube-system` ネームスペースにインストールし、
すべてのチャートで `global.platform=openshift` を設定する必要があります。

{{< tabset category-name="install-method" >}}

{{< tab name="Helm" category-value="helm" >}}

    インストールする**すべての**チャートで `--set global.platform=openshift` を指定してください。例：`istiod` チャート：

    {{< text syntax=bash >}}
    $ helm install istiod istio/istiod -n istio-system --set profile=ambient --set global.platform=openshift --wait
    {{< /text >}}

    また、`istio-cni` と `ztunnel` は `kube-system` ネームスペースにインストールしてください。例：

    {{< text syntax=bash >}}
    $ helm install istio-cni istio/cni -n kube-system --set profile=ambient --set global.platform=openshift --wait
    $ helm install ztunnel istio/ztunnel -n kube-system --set profile=ambient --set global.platform=openshift --wait
    {{< /text >}}

{{< /tab >}}

{{< tab name="istioctl" category-value="istioctl" >}}

    {{< text syntax=bash >}}
    $ istioctl install --set profile=openshift-ambient --skip-confirmation
    {{< /text >}}

{{< /tab >}}

{{< /tabset >}}

## CNI プラグイン {#cni-plugins}

一部の {{< gloss "CNI" >}}CNI プラグイン{{< /gloss >}} を使用する場合、以下の設定はすべてのプラットフォームに適用されます：

### Cilium

1. Cilium はデフォルトで他の CNI プラグインやその設定を積極的に削除します。
   チェーン利用を正しくサポートするには `cni.exclusive = false` を設定してください。
   詳細は [Cilium ドキュメント](https://docs.cilium.io/ja/stable/helm-reference/) を参照してください。
1. Cilium の BPF マスカレードは現在デフォルトで無効になっており、
   Istio がローカルリンク IP を使って Kubernetes のヘルスチェックを行う場合に問題があります。
   `bpf.masquerade=true` で BPF マスカレードを有効にすることは現在サポートされていません。
   これにより Istio Ambient の Pod ヘルスチェックが正しく動作しなくなります。
   Cilium のデフォルトの iptables マスカレード実装は引き続き動作します。
1. Cilium がノード ID の管理やノードレベルのヘルスプローブを Pod のホワイトリストに追加する方法のため、
   Ambient モードで Cilium CNI を使う場合、デフォルト DENY の `NetworkPolicy` を適用すると kubelet のヘルスチェック（Cilium はデフォルトで全てのポリシー適用を黙認）
   がブロックされます。これは、Istio が kubelet のヘルスチェックに Cilium で認識できないリンクローカル SNAT アドレスを使い、
   Cilium にはリンクローカルアドレスをポリシー適用から除外するオプションがないためです。

   以下の `CiliumClusterWideNetworkPolicy` を適用することで解決できます：

   {{< text syntax=yaml >}}
   apiVersion: "cilium.io/v2"
   kind: CiliumClusterwideNetworkPolicy
   metadata:
   name: "allow-ambient-hostprobes"
   spec:
   description: "Allows SNAT-ed kubelet health check probes into ambient pods"
   enableDefaultDeny:
   egress: false
   ingress: false
   endpointSelector: {}
   ingress:

   - fromCIDR: - "169.254.7.127/32"
     {{< /text >}}

   すでに他のデフォルト拒否の `NetworkPolicies` や `CiliumNetworkPolicies` を適用している場合を除き、このポリシーの上書きは不要です。

   詳細は [Issue #49277](https://github.com/istio/istio/issues/49277)
   および [CiliumClusterWideNetworkPolicy](https://docs.cilium.io/ja/stable/network/kubernetes/policy/#ciliumclusterwidenetworkpolicy) を参照してください。
