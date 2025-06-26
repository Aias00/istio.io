---
---
*   `kubectl` コマンドに `--context` パラメータを使用して、クラスター `cluster1` と `cluster2` にアクセスできます。
    例：`kubectl get pods --context cluster1`。
    以下のコマンドを使用して、コンテキストを一覧表示します：

    {{< text bash >}}
    $ kubectl config get-contexts
    CURRENT   NAME       CLUSTER    AUTHINFO       NAMESPACE
    *         cluster1   cluster1   user@foo.com   default
              cluster2   cluster2   user@foo.com   default
    {{< /text >}}

*   クラスターのコンテキストを環境変数に保存します：

    {{< text bash >}}
    $ export CTX_CLUSTER1=$(kubectl config view -o jsonpath='{.contexts[0].name}')
    $ export CTX_CLUSTER2=$(kubectl config view -o jsonpath='{.contexts[1].name}')
    $ echo "CTX_CLUSTER1 = ${CTX_CLUSTER1}, CTX_CLUSTER2 = ${CTX_CLUSTER2}"
    CTX_CLUSTER1 = cluster1, CTX_CLUSTER2 = cluster2
    {{< /text >}}

    {{< tip >}}
    2 つ以上のクラスターのコンテキストがあり、最初の 2 つ以外のクラスターを使用してメッシュを構成したい場合、手動で必要なコンテキスト名に環境変数を設定する必要があります。
    {{< /tip >}}
