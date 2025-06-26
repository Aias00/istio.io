---
---
## 開始する前に{#before-you-begin}

*   [インストールガイド](/ja/docs/setup/)の説明に従って Istio をインストールします。

    {{< tip >}}
    もし `demo` の[インストール設定](/ja/docs/setup/additional-setup/config-profiles/)を使用している場合、
    Egress Gateway とアクセスログが有効になります。
    {{< /tip >}}

*   [curl]({{< github_tree >}}/samples/curl) サンプルアプリケーションをデプロイして、リクエストを送信するテストソースとして使用します。
    もし[自動 Sidecar インジェクション](/ja/docs/setup/additional-setup/sidecar-injection/#automatic-sidecar-injection)を有効にしている場合、
    以下のコマンドを実行してサンプルアプリケーションをデプロイします：

    {{< text bash >}}
    $ kubectl apply -f @samples/curl/curl.yaml@
    {{< /text >}}

    そうでない場合、`curl` アプリケーションをデプロイする前に、手動で Sidecar をインジェクトします：

    {{< text bash >}}
    $ kubectl apply -f <(istioctl kube-inject -f @samples/curl/curl.yaml@)
    {{< /text >}}

    {{< tip >}}
    任意の `curl` がインストールされた Pod をテストソースとして使用できます。
    {{< /tip >}}

*   リクエストを送信するには、`SOURCE_POD` 環境変数を作成して、ソース Pod の名前を保存する必要があります：

    {{< text bash >}}
    $ export SOURCE_POD=$(kubectl get pod -l app=curl -o jsonpath={.items..metadata.name})
    {{< /text >}}
