---
title: Istio 証明書の有効期間を設定するにはどうすればよいですか？
weight: 70
---

Kubernetes で実行されているワークロードの Istio 証明書の有効期間は、デフォルトで 24 時間です。

[プロキシ設定](/ja/docs/reference/config/istio.mesh.v1alpha1/#ProxyConfig)の `proxyMetadata` フィールドをカスタマイズすることで、この設定をオーバーライドできます。例：

{{< text yaml >}}
proxyMetadata:
  SECRET_TTL: 48h
{{< /text >}}

{{< tip >}}
90 日を超える値は受け入れられません。
{{< /tip >}}
