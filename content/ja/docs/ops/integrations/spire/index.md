---
title: SPIRE
description: Istio を SPIRE と統合し、Envoy の SDS API を通じて暗号化 ID を取得する方法。
weight: 31
keywords: [kubernetes, spiffe, spire]
aliases:
owner: istio/wg-networking-maintainers
test: yes
---

[SPIRE](https://spiffe.io/docs/latest/spire-about/spire-concepts/) は SPIFFE 規格の本番環境向け実装であり、ノードやワークロードの認証を行い、異種環境で稼働するワークロードに安全な暗号化 ID を発行します。[Envoy の SDS API](https://www.envoyproxy.io/docs/envoy/latest/configuration/security/secret) と統合することで、SPIRE を Istio ワークロードの暗号化 ID ソースとして構成できます。Istio は Envoy SDS API を実装した UNIX ドメインソケットの存在を検出し、Envoy が直接通信して ID を取得できるようにします。

デフォルトの Istio ID 管理と比較して、SPIRE との統合は柔軟な認証オプションを提供します。SPIRE のプラグインアーキテクチャにより、Kubernetes のネームスペースやサービスアカウント認証以外の多様なワークロード認証が可能です。SPIRE のノード認証は、ワークロードが稼働する物理・仮想ハードウェアまで認証範囲を拡張します。

SPIRE と Istio の統合デモについては、[Envoy の SDS API を使った SPIRE の CA 統合]({{< github_tree >}}/samples/security/spire)を参照してください。

## SPIRE のインストール {#install-spire}

SPIRE のインストールと運用環境へのデプロイには、SPIRE のインストールガイドとベストプラクティスに従うことを推奨します。

本ガイドの例では、[SPIRE Helm Chart](https://artifacthub.io/packages/helm/spiffe/spire) を上流デフォルト値で使用し、SPIRE と Istio の統合に必要な設定のみに焦点を当てます。

{{< text syntax=bash snip_id=install_spire_crds >}}
$ helm upgrade --install -n spire-server spire-crds spire-crds --repo https://spiffe.github.io/helm-charts-hardened/ --create-namespace
{{< /text >}}

{{< text syntax=bash snip_id=install_spire_istio_overrides >}}
$ helm upgrade --install -n spire-server spire spire --repo https://spiffe.github.io/helm-charts-hardened/ --wait --set global.spire.trustDomain="example.org"
{{< /text >}}

{{< tip >}}
[SPIRE Helm Chart](https://artifacthub.io/packages/helm/spiffe/spire) のドキュメントも参照し、インストール時に設定できる追加値を確認してください。

重要：SPIRE と Istio で信頼ドメインを完全に一致させ、認証・認可エラーを防止し、[SPIFFE CSI ドライバー](https://github.com/spiffe/spiffe-csi) を有効化・インストールしてください。
{{< /tip >}}

上記の操作でデフォルトでインストールされるもの：

- [SPIFFE CSI ドライバー](https://github.com/spiffe/spiffe-csi)：Envoy 互換の SDS ソケットをプロキシにマウントします。Istio・SPIRE の両方で CSI ドライバーの利用が強く推奨されます。本ガイドも CSI ドライバー利用を前提とします。
- [SPIRE コントローラーマネージャー](https://github.com/spiffe/spire-controller-manager)：ワークロードの SPIFFE 登録作成を簡素化します。

## ワークロードの登録 {#register-workloads}

設計上、SPIRE は SPIRE サーバーに登録されたワークロードのみに ID を付与します。これにはユーザーワークロードと Istio コンポーネントが含まれます。SPIRE 統合を有効化した Istio Sidecar や Gateway は、事前に SPIRE 登録がなければ ID を取得できず、READY 状態になりません。

複数のセレクターで証明基準を強化する方法や利用可能なセレクターについては、[SPIRE ワークロード登録ドキュメント](https://spiffe.io/docs/latest/deploying/registering/)を参照してください。

このセクションでは、SPIRE サーバーで Istio ワークロードを登録する方法と例を紹介します。

{{< warning >}}
Istio ではワークロードの SPIFFE ID 形式が決まっています。すべての登録は `spiffe://<trust.domain>/ns/<namespace>/sa/<service-account>` 形式に従う必要があります。
{{< /warning >}}

### オプション 1：SPIRE コントローラーマネージャーによる自動登録 {#option-1-auto-registration-using-the-spire-controller-manager}

[ClusterSPIFFEID](https://github.com/spiffe/spire-controller-manager/blob/main/docs/clusterspiffeid-crd.md) カスタムリソースで定義したセレクターに一致する新しい Pod ごとに、自動的に新しい Entry が登録されます。

Istio Sidecar と Istio Gateway も SPIRE への登録が必要です。

#### Istio Gateway `ClusterSPIFFEID` {#istio-gateway-clusterspiffeid}

以下は、`istio-system` 名前空間にスケジューリングされ、サービスアカウントが `istio-ingressgateway-service-account` の Istio Ingress Gateway Pod を自動登録する `ClusterSPIFFEID` の例です。詳細は [SPIRE コントローラーマネージャードキュメント](https://github.com/spiffe/spire-controller-manager/blob/main/docs/clusterspiffeid-crd.md) を参照してください。

{{< text syntax=bash snip_id=spire_csid_istio_gateway >}}
$ kubectl apply -f - <<EOF
apiVersion: spire.spiffe.io/v1alpha1
kind: ClusterSPIFFEID
metadata:
name: istio-ingressgateway-reg
spec:
spiffeIDTemplate: "spiffe://{{ .TrustDomain }}/ns/{{ .PodMeta.Namespace }}/sa/{{ .PodSpec.ServiceAccountName }}"
workloadSelectorTemplates: - "k8s:ns:istio-system" - "k8s:sa:istio-ingressgateway-service-account"
EOF
{{< /text >}}

#### Istio Sidecar `ClusterSPIFFEID` {#istio-sidecar-clusterspiffeid}

以下は、`spiffe.io/spire-managed-identity: true` ラベル付きで `default` 名前空間にデプロイされた Pod を自動登録する `ClusterSPIFFEID` の例です。詳細は [SPIRE コントローラーマネージャードキュメント](https://github.com/spiffe/spire-controller-manager/blob/main/docs/clusterspiffeid-crd.md) を参照してください。

{{< text syntax=bash snip_id=spire_csid_istio_sidecar >}}
$ kubectl apply -f - <<EOF
apiVersion: spire.spiffe.io/v1alpha1
kind: ClusterSPIFFEID
metadata:
name: istio-sidecar-reg
spec:
spiffeIDTemplate: "spiffe://{{ .TrustDomain }}/ns/{{ .PodMeta.Namespace }}/sa/{{ .PodSpec.ServiceAccountName }}"
podSelector:
matchLabels:
spiffe.io/spire-managed-identity: "true"
workloadSelectorTemplates: - "k8s:ns:default"
EOF
{{< /text >}}

### オプション 2：手動登録 {#option-2-manual-registration}

SPIRE コントローラーマネージャーを使わずに手動で登録する場合は、[SPIRE の手動登録ドキュメント](https://spiffe.io/docs/latest/deploying/registering/)を参照してください。

以下は[オプション 1](#option-1-auto-registration-using-the-spire-controller-manager)の自動登録と同等の手動登録例です。SPIRE エージェントやノード証明の手動登録は[公式ドキュメント](https://spiffe.io/docs/latest/deploying/registering/#1-defining-the-spiffe-id-of-the-agent)を参照してください。

1. `spire-server` Pod を取得：

   {{< text syntax=bash snip_id=set_spire_server_pod_name_var >}}
   $ SPIRE_SERVER_POD=$(kubectl get pod -l statefulset.kubernetes.io/pod-name=spire-server-0 -n spire-server -o jsonpath="{.items[0].metadata.name}")
   {{< /text >}}

1. Istio Ingress Gateway Pod の Entry を登録：

   {{< text bash >}}
   $ kubectl exec -n spire "$SPIRE_SERVER_POD" -- \
   /opt/spire/bin/spire-server entry create \
    -spiffeID spiffe://example.org/ns/istio-system/sa/istio-ingressgateway-service-account \
    -parentID spiffe://example.org/ns/spire/sa/spire-agent \
    -selector k8s:sa:istio-ingressgateway-service-account \
    -selector k8s:ns:istio-system \
    -socketPath /run/spire/sockets/server.sock

   Entry ID : 6f2fe370-5261-4361-ac36-10aae8d91ff7
   SPIFFE ID : spiffe://example.org/ns/istio-system/sa/istio-ingressgateway-service-account
   Parent ID : spiffe://example.org/ns/spire/sa/spire-agent
   Revision : 0
   TTL : default
   Selector : k8s:ns:istio-system
   Selector : k8s:sa:istio-ingressgateway-service-account
   {{< /text >}}

1. Istio Sidecar 注入ワークロードの Entry を登録：

   {{< text bash >}}
   $ kubectl exec -n spire "$SPIRE_SERVER_POD" -- \
   /opt/spire/bin/spire-server entry create \
    -spiffeID spiffe://example.org/ns/default/sa/curl \
    -parentID spiffe://example.org/ns/spire/sa/spire-agent \
    -selector k8s:ns:default \
    -selector k8s:pod-label:spiffe.io/spire-managed-identity:true \
    -socketPath /run/spire/sockets/server.sock
   {{< /text >}}

## Istio のインストール {#install-istio}

1. [Istio リリースをダウンロード](/ja/docs/setup/additional-setup/download-istio-release/)します。

1. Ingress Gateway と `istio-proxy` 用にカスタムパッチで Istio 設定を作成します。Ingress Gateway には `spiffe.io/spire-managed-identity: "true"` ラベルを付与します。

   {{< text syntax=bash snip_id=define_istio_operator_for_auto_registration >}}
   $ cat <<EOF > ./istio.yaml
   apiVersion: install.istio.io/v1alpha1
   kind: IstioOperator
   metadata:
   namespace: istio-system
   spec:
   profile: default
   meshConfig:
   trustDomain: example.org
   values: # これは Sidecar テンプレートのカスタマイズ用です。 # SPIRE でこの Pod の ID を管理することを示すラベルと、CSI ドライバーのマウントを追加します。
   sidecarInjectorWebhook:
   templates:
   spire: |
   labels:
   spiffe.io/spire-managed-identity: "true"
   spec:
   containers: - name: istio-proxy
   volumeMounts: - name: workload-socket
   mountPath: /run/secrets/workload-spiffe-uds
   readOnly: true
   volumes: - name: workload-socket
   csi:
   driver: "csi.spiffe.io"
   readOnly: true
   components:
   ingressGateways: - name: istio-ingressgateway
   enabled: true
   label:
   istio: ingressgateway
   k8s:
   overlays: # これは Ingress Gateway テンプレートのカスタマイズ用です。 # CSI ドライバーのマウントと、CSI ソケットができるまで起動を待つ init コンテナを追加します。 - apiVersion: apps/v1
   kind: Deployment
   name: istio-ingressgateway
   patches: - path: spec.template.spec.volumes.[name:workload-socket]
   value:
   name: workload-socket
   csi:
   driver: "csi.spiffe.io"
   readOnly: true - path: spec.template.spec.containers.[name:istio-proxy].volumeMounts.[name:workload-socket]
   value:
   name: workload-socket
   mountPath: "/run/secrets/workload-spiffe-uds"
   readOnly: true - path: spec.template.spec.initContainers
   value: - name: wait-for-spire-socket
   image: busybox:1.36
   volumeMounts: - name: workload-socket
   mountPath: /run/secrets/workload-spiffe-uds
   readOnly: true
   env: - name: CHECK_FILE
   value: /run/secrets/workload-spiffe-uds/socket
   command: - sh - "-c" - |-
   echo "$(date -Iseconds)" Waiting for: ${CHECK_FILE}
                              while [[ ! -e ${CHECK_FILE} ]] ; do
                                echo "$(date -Iseconds)" File does not exist: ${CHECK_FILE}
   sleep 15
   done
   ls -l ${CHECK_FILE}
   EOF
   {{< /text >}}

1. 設定を適用：

   {{< text syntax=bash snip_id=apply_istio_operator_configuration >}}
   $ istioctl install --skip-confirmation -f ./istio.yaml
   {{< /text >}}

1. Ingress Gateway Pod の状態を確認：

   {{< text syntax=bash snip_id=none >}}
   $ kubectl get pods -n istio-system
   NAME READY STATUS RESTARTS AGE
   istio-ingressgateway-5b45864fd4-lgrxs 1/1 Running 0 17s
   istiod-989f54d9c-sg7sn 1/1 Running 0 23s
   {{< /text >}}

   Ingress Gateway Pod は `Ready` となり、SPIRE サーバーで自動的に登録エントリが作成されます。Envoy は SPIRE から暗号化 ID を取得できます。

   この構成では、SPIRE の UNIX ドメインソケットが作成されるまで Ingress Gateway の起動を待つ `initContainer` も追加されています。SPIRE エージェントが準備できていない場合や、ソケットパスが一致しない場合、Ingress Gateway の `initContainer` は永遠に待機します。

1. サンプルワークロードをデプロイ：

   {{< text syntax=bash snip_id=apply_curl >}}
   $ istioctl kube-inject --filename @samples/security/spire/curl-spire.yaml@ | kubectl apply -f -
   {{< /text >}}

   `spiffe.io/spire-managed-identity` ラベルの付与に加え、SPIFFE CSI ドライバーのボリュームで SPIRE エージェントソケットにアクセスできる必要があります。これには[Istio のインストール](#install-istio)セクションの `spire` Pod アノテーションテンプレートを使うか、ワークロードの Deployment 仕様に CSI ボリュームを追加します。以下の例で両方の方法を示します：

   {{< text syntax=yaml snip_id=none >}}
   apiVersion: apps/v1
   kind: Deployment
   metadata:
   name: curl
   spec:
   replicas: 1
   selector:
   matchLabels:
   app: curl
   template:
   metadata:
   labels:
   app: curl # カスタム Sidecar テンプレートを注入
   annotations:
   inject.istio.io/templates: "sidecar,spire"
   spec:
   terminationGracePeriodSeconds: 0
   serviceAccountName: curl
   containers: - name: curl
   image: curlimages/curl
   command: ["/bin/sleep", "3650d"]
   imagePullPolicy: IfNotPresent
   volumeMounts: - name: tmp
   mountPath: /tmp
   securityContext:
   runAsUser: 1000
   volumes: - name: tmp
   emptyDir: {} # CSI ボリューム - name: workload-socket
   csi:
   driver: "csi.spiffe.io"
   readOnly: true
   {{< /text >}}

Istio の設定は Ingress Gateway と Sidecar 注入ワークロードの両方で `spiffe-csi-driver` を共有し、SPIRE エージェントの UNIX ドメインソケットへのアクセス権を付与します。

[ワークロードの ID が作成されたかの検証](#verifying-that-identities-were-created-for-workloads)も参照してください。

## ワークロード ID の作成確認 {#verifying-that-identities-were-created-for-workloads}

以下のコマンドでワークロードの ID が作成されたか確認できます：

{{< text syntax=bash snip_id=none >}}
$ kubectl exec -t "$SPIRE_SERVER_POD" -n spire-server -c spire-server -- ./bin/spire-server entry show
Found 2 entries
Entry ID : c8dfccdc-9762-4762-80d3-5434e5388ae7
SPIFFE ID : spiffe://example.org/ns/istio-system/sa/istio-ingressgateway-service-account
Parent ID : spiffe://example.org/spire/agent/k8s_psat/demo-cluster/bea19580-ae04-4679-a22e-472e18ca4687
Revision : 0
X509-SVID TTL : default
JWT-SVID TTL : default
Selector : k8s:pod-uid:88b71387-4641-4d9c-9a89-989c88f7509d

Entry ID : af7b53dc-4cc9-40d3-aaeb-08abbddd8e54
SPIFFE ID : spiffe://example.org/ns/default/sa/curl
Parent ID : spiffe://example.org/spire/agent/k8s_psat/demo-cluster/bea19580-ae04-4679-a22e-472e18ca4687
Revision : 0
X509-SVID TTL : default
JWT-SVID TTL : default
Selector : k8s:pod-uid:ee490447-e502-46bd-8532-5a746b0871d6
{{< /text >}}

Ingress-gateway Pod の状態確認：

{{< text syntax=bash snip_id=none >}}
$ kubectl get pods -n istio-system
NAME READY STATUS RESTARTS AGE
istio-ingressgateway-5b45864fd4-lgrxs 1/1 Running 0 60s
istiod-989f54d9c-sg7sn 1/1 Running 0 45s
{{< /text >}}

Ingress Gateway Pod の登録後、Envoy は SPIRE から発行された ID を受け取り、すべての TLS/mTLS 通信に利用します。

### ワークロード ID が SPIRE から発行されたかの確認 {#check-that-the-workload-identity-was-issued-by-spire}

1. Pod 情報を取得：

   {{< text syntax=bash snip_id=set_curl_pod_var >}}
   $ CURL_POD=$(kubectl get pod -l app=curl -o jsonpath="{.items[0].metadata.name}")
   {{< /text >}}

1. `istioctl proxy-config secret` コマンドで curl の SVID ID ドキュメントを取得：

   {{< text syntax=bash snip_id=get_curl_svid >}}
   $ istioctl proxy-config secret "$CURL_POD" -o json | jq -r \
   '.dynamicActiveSecrets[0].secret.tlsCertificate.certificateChain.inlineBytes' | base64 --decode > chain.pem
   {{< /text >}}

1. 証明書を確認し、SPIRE が発行者であることを確認：

   {{< text syntax=bash snip_id=get_svid_subject >}}
   $ openssl x509 -in chain.pem -text | grep SPIRE
   Subject: C = US, O = SPIRE, CN = curl-5f4d47c948-njvpk
   {{< /text >}}

## SPIFFE フェデレーション {#spiffe-federation}

SPIRE サーバーは異なる信頼ドメインの SPIFFE ID を認証できます（SPIFFE フェデレーション）。

SPIRE エージェントを Envoy SDS API 経由で Envoy に連携させることで、Envoy は[バリデーションコンテキスト](https://spiffe.io/docs/latest/microservices/envoy/#validation-context)を使って他ドメインのワークロード証明書を検証し、信頼できます。Istio で SPIRE 統合による SPIFFE ID フェデレーションを有効にするには、[SPIRE エージェント SDS 設定](https://github.com/spiffe/spire/blob/main/doc/spire_agent.md#sds-configuration)を参照し、以下の SDS 設定値を構成してください。

| 設定                       | 説明                                                                                      | リソース名 |
| -------------------------- | ----------------------------------------------------------------------------------------- | ---------- |
| `default_svid_name`        | Envoy SDS のデフォルト `X509-SVID` の TLS 証明書リソース名                                | デフォルト |
| `default_bundle_name`      | Envoy SDS のデフォルト X.509 bundle のバリデーションコンテキストリソース名                | 空         |
| `default_all_bundles_name` | Envoy SDS のすべての bundle（フェデレーション含む）のバリデーションコンテキストリソース名 | ROOTCA     |

これにより、Envoy は SPIRE から直接フェデレーション bundle を取得できます。

### フェデレーション登録エントリの作成 {#create-federated-registration-entries}

- SPIRE コントローラーマネージャーを使う場合は、[ClusterSPIFFEID CR](https://github.com/spiffe/spire-controller-manager/blob/main/docs/clusterspiffeid-crd.md) の `federatesWith` フィールドに連携したい信頼ドメインを指定してください：

  {{< text syntax=yaml snip_id=none >}}
  apiVersion: spire.spiffe.io/v1alpha1
  kind: ClusterSPIFFEID
  metadata:
  name: federation
  spec:
  spiffeIDTemplate: "spiffe://{{ .TrustDomain }}/ns/{{ .PodMeta.Namespace }}/sa/{{ .PodSpec.ServiceAccountName }}"
  podSelector:
  matchLabels:
  spiffe.io/spire-managed-identity: "true"
  federatesWith: ["example.io", "example.ai"]
  {{< /text >}}

- 手動登録の場合は、[フェデレーション用登録エントリの作成](https://spiffe.io/docs/latest/architecture/federation/readme/#create-registration-entries-for-federation)を参照してください。

## SPIRE のクリーンアップ {#cleanup-spire}

Helm Chart をアンインストールして SPIRE を削除します：

{{< text syntax=bash snip_id=uninstall_spire >}}
$ helm delete -n spire-server spire
{{< /text >}}

{{< text syntax=bash snip_id=uninstall_spire_crds >}}
$ helm delete -n spire-server spire-crds
{{< /text >}}
