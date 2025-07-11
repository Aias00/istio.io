---
title: バグ報告
description: もし問題を発見したらどうすればよいか。
weight: 34
aliases:
  - /zh/bugs.html
  - /zh/bugs/index.html
  - /zh/help/bugs/
  - /zh/about/bugs
  - /zh/latest/about/bugs
icon: bugs
owner: istio/wg-docs-maintainers
test: n/a
---

あらら！問題を発見しましたか？ぜひお知らせください。

## 製品のバグ {#product-bugs}

[課題データベース](https://github.com/istio/istio/issues/)で、すでにその問題が知られているか、いつ修正されるかを確認してください。
もしデータベースに見つからない場合は、[新しい Issue](https://github.com/istio/istio/issues/new/choose) を作成して、どんなバグが発生したか教えてください。

もしバグがセキュリティ脆弱性であると思われる場合は、[セキュリティ脆弱性の報告](/ja/about/security-vulnerabilities/)をご覧ください。

### Kubernetes クラスターの状態アーカイブ {#Kubernetes-cluster-state-archives}

Kubernetes をご利用の場合は、バグ報告時にクラスターの状態アーカイブを添付することを検討してください。
簡単にアーカイブを作成するには、`istioctl bug-report` コマンドを実行して、Kubernetes クラスター内の関連状態を含むアーカイブを生成できます：

{{< text bash >}}
$ istioctl bug-report
{{< /text >}}

バグ報告時に、生成された `bug-report.tgz` を添付してください。

メッシュが複数のクラスターにまたがる場合は、各クラスターで `istioctl bug-report` を実行し、`--context` または `--kubeconfig` オプションを指定してください。

{{< tip >}}
`istioctl bug-report` コマンドは `istioctl` 1.8.0 以降で利用可能ですが、より古いバージョンの Istio にも利用できます。
{{< /tip >}}

{{< tip >}}
大規模なクラスターで `bug-report` を実行すると、完了しない場合があります。
その場合は `--include ns1,ns2` オプションを使って、関連する名前空間のみのプロキシコマンドやログ収集を行ってください。その他の `bug-report` オプションについては、[istioctl bug-report リファレンス](/ja/docs/reference/commands/istioctl/#istioctl-bug-report) を参照してください。
{{< /tip >}}

`bug-report` コマンドが使えない場合は、以下の情報を含む独自のアーカイブを添付してください：

- `istioctl analyze` の出力：

  {{< text bash >}}
  $ istioctl analyze --all-namespaces
  {{< /text >}}

- すべての名前空間の `pods`、`services`、`deployments`、`endpoints` リソース：

  {{< text bash >}}
  $ kubectl get pods,services,deployments,endpoints --all-namespaces -o yaml > k8s_resources.yaml
  {{< /text >}}

- `istio-system` 名前空間の Secret：

  {{< text bash >}}
  $ kubectl --namespace istio-system get secrets
  {{< /text >}}

- `istio-system` 名前空間の ConfigMap：

  {{< text bash >}}
  $ kubectl --namespace istio-system get cm -o yaml
  {{< /text >}}

- すべての Istio コンポーネントおよびサイドカーの現在と過去のログ。以下はログ取得例です。環境に合わせて調整してください：

  - Istiod のログ：

    {{< text bash >}}
    $ kubectl logs -n istio-system -l app=istiod
    {{< /text >}}

  - Ingress Gateway のログ：

    {{< text bash >}}
    $ kubectl logs -l istio=ingressgateway -n istio-system
    {{< /text >}}

  - Egress Gateway のログ：

    {{< text bash >}}
    $ kubectl logs -l istio=egressgateway -n istio-system
    {{< /text >}}

  - サイドカーのログ：

    {{< text bash >}}
    $ for ns in $(kubectl get ns -o jsonpath='{.items[*].metadata.name}') ; do kubectl logs -l service.istio.io/canonical-revision -c istio-proxy -n $ns ; done
    {{< /text >}}

- すべての Istio 関連の設定ファイル：

  {{< text bash >}}
  $ kubectl get istio-io --all-namespaces -o yaml
  {{< /text >}}

## ドキュメントのバグ {#documentation-bugs}

[ドキュメント課題データベース](https://github.com/istio/istio.io/issues/)で、すでにその問題が知られているか、いつ修正されるかを確認してください。
見つからない場合は、[こちらで課題を報告](https://github.com/istio/istio.io/issues/new)してください。
ページの右下にある「GitHub でこのページを編集」リンクから、修正提案を送ることもできます。
