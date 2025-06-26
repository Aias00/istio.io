---
---
Istio をアップグレードする前に、カスタム設定のバックアップを作成し、必要に応じてそのバックアップから復元することをお勧めします：

{{< text bash >}}
$ kubectl get istio-io --all-namespaces -oyaml > "$HOME"/istio_resource_backup.yaml
{{< /text >}}

カスタム設定を復元するには、以下のようにします：

{{< text bash >}}
$ kubectl apply -f "$HOME"/istio_resource_backup.yaml
{{< /text >}}
