---
---
{{< warning >}}
Helm から Istio 1.23 またはそれ以前のバージョンの CRD をアップグレードする場合、以下のようなエラーが発生する可能性があります：

`Error: rendered manifests contain a resource that already exists. Unable to continue with update: CustomResourceDefinition "wasmplugins.extensions.istio.io" in namespace "" exists and cannot be imported into the current release: invalid ownership metadata`

以下の `kubectl` コマンドを使用して、一括移行でこの問題を解決できます：

{{< text syntax=bash snip_id=adopt_legacy_crds >}}
$ for crd in $(kubectl get crds -l chart=istio -o name && kubectl get crds -l app.kubernetes.io/part-of=istio -o name)
$ do
$    kubectl label "$crd" "app.kubernetes.io/managed-by=Helm"
$    kubectl annotate "$crd" "meta.helm.sh/release-name=istio-base" # ドキュメントのデフォルト値と異なる場合は、実際の Helm バージョン名に置き換えてください
$    kubectl annotate "$crd" "meta.helm.sh/release-namespace=istio-system" # 実際の istio 名前空間に置き換えてください
$ done
{{< /text >}}

{{< /warning >}}
