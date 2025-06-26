---
---
OpenTelemetry Collector の名前空間を作成します：

{{< text bash >}}
$ kubectl apply -f @samples/open-telemetry/otel.yaml@ -n istio-system
$ kubectl create namespace observability
{{< /text >}}

OpenTelemetry Collector をデプロイします。
[このサンプル設定]({{< github_blob >}}/samples/open-telemetry/otel.yaml)を起点として使用できます。

{{< text bash >}}
$ kubectl apply -f @samples/open-telemetry/otel.yaml@ -n observability
{{< /text >}}
