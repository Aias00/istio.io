---
---
すべての Istio CRD の作成が完了するのを待ちます：

{{< text bash >}}
$ kubectl -n istio-system wait --for=condition=complete job --all
{{< /text >}}
