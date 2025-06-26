---
---
Kubernetes Gateway API CRD は、多くの Kubernetes クラスターではデフォルトでインストールされていないため、
Gateway API を使用する前に、これらの CRD がインストールされていることを確認してください：

{{< text syntax=bash snip_id=install_crds >}}
$ kubectl get crd gateways.gateway.networking.k8s.io &> /dev/null || \
  kubectl apply -f https://github.com/kubernetes-sigs/gateway-api/releases/download/{{< k8s_gateway_api_version >}}/standard-install.yaml
{{< /text >}}
