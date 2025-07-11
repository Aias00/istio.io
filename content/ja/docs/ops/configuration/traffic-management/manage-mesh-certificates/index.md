---
title: メッシュ内証明書の管理
linktitle: メッシュ内証明書の管理
description: メッシュ内の証明書を設定する方法。
weight: 30
keywords: [traffic-management, proxy]
owner: istio/wg-networking-maintainers,istio/wg-environments-maintainers
test: n/a
---

{{< boilerplate experimental >}}

多くのユーザーは、環境で使用するさまざまな証明書を管理する必要があります。
たとえば、楕円曲線暗号（ECC）を使用したいユーザーもいれば、より大きなビット長の RSA 証明書を必要とするユーザーもいます。
多くのユーザーにとって、環境で証明書を設定するのは困難な作業かもしれません。

本内容はメッシュ内部通信にのみ適用されます。ゲートウェイ上の証明書を管理するには、
[セキュアゲートウェイ](/ja/docs/tasks/traffic-management/ingress/secure-ingress/)のドキュメントを参照してください。
istiod がワークロード証明書を発行するために使用する CA の管理については、
[プラグイン CA 証明書](/ja/docs/tasks/security/cert-management/plugin-ca-cert/)のドキュメントを参照してください。

## istiod

Istio をルート CA 証明書なしでインストールした場合、istiod は RSA 2048 で自己署名 CA 証明書を生成します。

自己署名 CA 証明書のビット長を変更するには、`istioctl` に渡す IstioOperator
マニフェストを修正するか、Helm で [istio-discovery]({{< github_tree >}}/manifests/charts/istio-control/istio-discovery) チャートをインストールする際に values ファイルを使用してください。

{{< tip >}}
[pilot-discovery](/ja/docs/reference/commands/pilot-discovery/) には多くの環境変数がありますが、
ここではその一部のみを紹介します。
{{< /tip >}}

{{< tabset category-name="証明書" >}}

{{< tab name="IstioOperator" category-value="iop" >}}

{{< text yaml >}}
apiVersion: install.istio.io/v1alpha1
kind: IstioOperator
spec:
values:
pilot:
env:
CITADEL_SELF_SIGNED_CA_RSA_KEY_SIZE: 4096
{{< /text >}}

{{< /tab >}}

{{< tab name="Helm" category-value="helm" >}}

{{< text yaml >}}
pilot:
env:
CITADEL_SELF_SIGNED_CA_RSA_KEY_SIZE: 4096
{{< /text >}}

{{< /tab >}}

{{< /tabset >}}

## Sidecar

Sidecar はメッシュ内部通信に使用する証明書を自身で管理するため、
Sidecar は自身の秘密鍵と生成された証明書署名要求（CSR）を管理します。
これには、Sidecar インジェクターで環境変数を注入する必要があります。

{{< tip >}}
[pilot-agent](/ja/docs/reference/commands/pilot-agent/) には多くの環境変数がありますが、
ここではその一部のみを紹介します。
{{< /tip >}}

{{< tabset category-name="gateway-install-type" >}}

{{< tab name="IstioOperator" category-value="iop" >}}

{{< text yaml >}}
apiVersion: install.istio.io/v1alpha1
kind: IstioOperator
spec:
meshConfig:
defaultConfig:
proxyMetadata:
CITADEL_SELF_SIGNED_CA_RSA_KEY_SIZE: 4096
{{< /text >}}

{{< /tab >}}

{{< tab name="Helm" category-value="helm" >}}

{{< text yaml >}}
meshConfig:
defaultConfig:
proxyMetadata:
CITADEL_SELF_SIGNED_CA_RSA_KEY_SIZE: 4096
{{< /text >}}

{{< /tab >}}

{{< tab name="Annotation" category-value="annotation" >}}

{{< text yaml >}}
apiVersion: apps/v1
kind: Deployment
metadata:
name: curl
spec:
...
template:
metadata:
...
annotations:
...
proxy.istio.io/config: |
CITADEL_SELF_SIGNED_CA_RSA_KEY_SIZE: 4096
spec:
...
{{< /text >}}

{{< /tab >}}

{{< /tabset >}}

### 署名アルゴリズム {#signature-algorithm}

デフォルトでは、Sidecar は RSA 証明書を作成します。
ECC に変更したい場合は、`ECC_SIGNATURE_ALGORITHM` を `ECDSA` に設定してください。

{{< tabset category-name="gateway-install-type" >}}

{{< tab name="IstioOperator" category-value="iop" >}}

{{< text yaml >}}
apiVersion: install.istio.io/v1alpha1
kind: IstioOperator
spec:
meshConfig:
defaultConfig:
proxyMetadata:
ECC_SIGNATURE_ALGORITHM: "ECDSA"
{{< /text >}}

{{< /tab >}}

{{< tab name="Helm" category-value="helm" >}}

{{< text yaml >}}
meshConfig:
defaultConfig:
proxyMetadata:
ECC_SIGNATURE_ALGORITHM: "ECDSA"
{{< /text >}}

{{< /tab >}}

{{< /tabset >}}

`ECC_CURVE` でサポートされるのは P256 と P384 のみです。

RSA 署名アルゴリズムを維持したまま RSA キーサイズを変更したい場合は、
`WORKLOAD_RSA_KEY_SIZE` の値を変更してください。
