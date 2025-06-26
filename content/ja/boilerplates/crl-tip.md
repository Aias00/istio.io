---
---
{{< tip >}}
必要に応じて、証明書失効リスト (CRL) を含めることができます。
これは、[証明書失効リスト (CRL)](https://datatracker.ietf.org/doc/html/rfc5280) として使用され、`ca.crl` というキー名で提供されます。
その場合、上記の例に別のパラメータを追加して CRL を提供します：`--from-file=ca.crl=/some/path/to/your-crl.pem`。
{{< /tip >}}
