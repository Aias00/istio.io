---
title: Kops
description: Istio 用の Kops セットアップ手順。
weight: 33
skip_seealso: true
keywords: [platform-setup, kubernetes, kops]
owner: istio/wg-environments-maintainers
test: no
---

{{< tip >}}
Kubernetes クラスタ 1.22 以降で Istio を実行する場合、特別な設定は不要です。以前の Kubernetes バージョンの場合は、以下の手順を続けてください。
{{< /tip >}}

Kops 管理クラスタ上で Istio の [Secret Discovery Service](https://www.envoyproxy.io/docs/envoy/latest/configuration/security/secret#sds-configuration) (SDS) をメッシュで利用したい場合、
[追加設定](https://kubernetes.io/ja/docs/tasks/configure-pod-container/configure-service-account/#service-account-token-volume-projection) を行い、API サーバーでサービスアカウントトークンのボリューム投影を有効にする必要があります。

1. 設定ファイルを開きます：

   {{< text bash >}}
   $ kops edit cluster $YOURCLUSTER
   {{< /text >}}

1. 設定ファイルに次の内容を追加します：

   {{< text yaml >}}
   kubeAPIServer:
   apiAudiences: - api - istio-ca
   serviceAccountIssuer: kubernetes.default.svc
   {{< /text >}}

1. 次のコマンドで更新を適用します：

   {{< text bash >}}
   $ kops update cluster
   $ kops update cluster --yes
   {{< /text >}}

1. ローリングアップデートを実行します：

   {{< text bash >}}
   $ kops rolling-update cluster
   $ kops rolling-update cluster --yes
   {{< /text >}}
