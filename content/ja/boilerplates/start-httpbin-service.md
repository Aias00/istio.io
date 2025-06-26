---
---
*   [Httpbin]({{< github_tree >}}/samples/httpbin) サンプルプログラムを起動します。

    [Sidecar 自動注入](/ja/docs/setup/additional-setup/sidecar-injection/#automatic-sidecar-injection)が有効な場合、`httpbin` サービスを以下のコマンドでデプロイします：

    {{< text bash >}}
    $ kubectl apply -f @samples/httpbin/httpbin.yaml@
    {{< /text >}}

    そうでない場合、`httpbin` アプリケーションをデプロイする前に手動で注入する必要があります。デプロイコマンドは次のとおりです：

    {{< text bash >}}
    $ kubectl apply -f <(istioctl kube-inject -f @samples/httpbin/httpbin.yaml@)
    {{< /text >}}
