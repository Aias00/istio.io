---
---
{{< tip >}}
Istio はデフォルトで `auto_sni` と `auto_san_validation` を有効にしています。
これは、`DestinationRule` で `sni` が明示的に設定されていない限り、新しい上流接続のトランスポートソケットの SNI が、下流の HTTP ホスト/認証ヘッダーに基づいて設定されることを意味します。
`sni` が設定されていない場合でも、`DestinationRule` で `subjectAltNames` が設定されていない場合、
`auto_san_validation` が有効になり、新しい上流接続の上流が提示する証明書が、下流の HTTP ホスト/認証ヘッダーに基づいて自動的に検証されます。
{{< /tip >}}
