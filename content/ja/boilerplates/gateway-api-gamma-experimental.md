---
---
{{< tip >}}
{{< boilerplate gateway-api-future >}}
{{< boilerplate gateway-api-choose >}}
{{< /tip >}}

{{< warning >}}
このドキュメントは[実験的](https://gateway-api.sigs.k8s.io/geps/overview/#status)
Gateway API を使用して Istio を設定する前に、以下を確認してください：

1) **実験版** の Gateway API CRD をインストールします：

    {{< text syntax=bash snip_id=install_experimental_crds >}}
    $ kubectl kustomize "github.com/kubernetes-sigs/gateway-api/config/crd/experimental?ref={{< k8s_gateway_api_version >}}" | kubectl apply -f -
    {{< /text >}}

2) Istio をインストールする際に、`PILOT_ENABLE_ALPHA_GATEWAY_API` 環境変数を `true` に設定して、Istio が Alpha バージョンの Gateway API リソースを読み取るようにします：

    {{< text syntax=bash snip_id=enable_alpha_crds >}}
    $ istioctl install --set values.pilot.env.PILOT_ENABLE_ALPHA_GATEWAY_API=true --set profile=minimal -y
    {{< /text >}}

{{< /warning >}}
