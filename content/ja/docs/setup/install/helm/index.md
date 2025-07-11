---
title: Helm を使用したインストール
linktitle: Helm を使用したインストール
description: Helm を使用して K8s クラスターに Istio をインストールおよび設定します。
weight: 30
keywords: [kubernetes, helm]
owner: istio/wg-environments-maintainers
test: yes
---

[Helm](https://helm.sh/docs/) を使用して Istio メッシュをインストールおよび設定するには、このガイドに従ってください。

{{< boilerplate helm-preamble >}}

{{< boilerplate helm-prereqs >}}

## インストール手順 {#installation-steps}

このセクションでは、Helm を使用して Istio をインストールするプロセスについて説明します。Helm インストールの一般的な構文は次のとおりです：

{{< text syntax=bash snip_id=none >}}
$ helm install <release> <chart> --namespace <namespace> --create-namespace [--set <other_parameters>]
{{< /text >}}

このコマンドで指定される変数は以下のとおりです：

- `<chart>`：パッケージ化された Chart のパス、または未パッケージの Chart ディレクトリまたは URL。
- `<release>`：インストール後の Helm Chart を識別および管理するための名前。
- `<namespace>`：Chart をインストールする名前空間。

1 つ以上の `--set <parameter>=<value>` パラメーターを使用してデフォルトの設定値を変更できます。
または、`--values <file>` パラメーターを使用してカスタム値ファイルで複数のパラメーターを指定できます。

{{< tip >}}
`helm show values <chart>` コマンドを使用して設定パラメーターのデフォルト値を表示するか、`artifacthub` Chart
ドキュメントの[カスタムリソースパラメーター](https://artifacthub.io/packages/helm/istio-official/base?modal=values)、
[Istiod Chart 設定パラメーター](https://artifacthub.io/packages/helm/istio-official/istiod?modal=values)、
[Gateway Chart 設定パラメーター](https://artifacthub.io/packages/helm/istio-official/gateway?modal=values)を参照してください。
{{< /tip >}}

1. Istio Base Chart をインストールします。これには、Istio コントロールプレーンをデプロイする前にインストールする必要があるクラスター全体のカスタムリソース定義（CRD）が含まれています：

   {{< warning >}}
   リビジョンインストールを実行する場合、Base Chart には `--set defaultRevision=<revision>` 値を設定してリソース検証を機能させる必要があります。
   以下では `default` リビジョンをインストールするため、`--set defaultRevision=default` パラメーターを設定します。
   {{< /warning >}}

   {{< text syntax=bash snip_id=install_base >}}
   $ helm install istio-base istio/base -n istio-system --set defaultRevision=default --create-namespace
   {{< /text >}}

1. `helm ls` コマンドを使用して CRD のインストールを確認します：

   {{< text syntax=bash >}}
   $ helm ls -n istio-system
   NAME NAMESPACE REVISION UPDATED STATUS CHART APP VERSION
   istio-base istio-system 1 2024-04-17 22:14:45.964722028 +0000 UTC deployed base-{{< istio_full_version >}} {{< istio_full_version >}}
   {{< /text >}}

   出力で `istio-base` のエントリを見つけて、ステータスが `deployed` に設定されていることを確認してください。

1. Istio CNI Chart を使用する予定の場合は、今すぐ行う必要があります。
   詳細については、[CNI プラグインによる Istio のインストール](/ja/docs/setup/additional-setup/cni/#installing-with-helm)を参照してください。

1. `istiod` サービスをデプロイする Istio Discovery Chart をインストールします：

   {{< text syntax=bash snip_id=install_discovery >}}
   $ helm install istiod istio/istiod -n istio-system --wait
   {{< /text >}}

1. Istio Discovery Chart のインストールを確認します：

   {{< text syntax=bash >}}
   $ helm ls -n istio-system
   NAME NAMESPACE REVISION UPDATED STATUS CHART APP VERSION
   istio-base istio-system 1 2024-04-17 22:14:45.964722028 +0000 UTC deployed base-{{< istio_full_version >}} {{< istio_full_version >}}
   istiod istio-system 1 2024-04-17 22:14:45.964722028 +0000 UTC deployed istiod-{{< istio_full_version >}} {{< istio_full_version >}}
   {{< /text >}}

1. インストールされた Helm Chart のステータスを取得して、デプロイされていることを確認します：

   {{< text syntax=bash >}}
   $ helm status istiod -n istio-system
   NAME: istiod
   LAST DEPLOYED: Fri Jan 20 22:00:44 2023
   NAMESPACE: istio-system
   STATUS: deployed
   REVISION: 1
   TEST SUITE: None
   NOTES:
   "istiod" successfully installed!

   To learn more about the release, try:
   $ helm status istiod
   $ helm get all istiod

   Next steps:

   - Deploy a Gateway: https://istio.io/latest/docs/setup/additional-setup/gateway/
   - Try out our tasks to get started on common configurations:
     - https://istio.io/latest/docs/tasks/traffic-management
     - https://istio.io/latest/docs/tasks/security/
     - https://istio.io/latest/docs/tasks/policy-enforcement/
     - https://istio.io/latest/docs/tasks/policy-enforcement/
   - Review the list of actively supported releases, CVE publications and our hardening guide:
     - https://istio.io/latest/docs/releases/supported-releases/
     - https://istio.io/latest/news/security/
     - https://istio.io/latest/docs/ops/best-practices/security/

   For further documentation see https://istio.io website

   Tell us how your install/upgrade experience went at https://forms.gle/99uiMML96AmsXY5d6
   {{< /text >}}

1. `istiod` サービスが正常にインストールされたことを確認し、その Pod が実行中であることを確認します:    

   {{< text syntax=bash >}}
   $ kubectl get deployments -n istio-system --output wide
   NAME READY UP-TO-DATE AVAILABLE AGE CONTAINERS IMAGES SELECTOR
   istiod 1/1 1 1 10m discovery docker.io/istio/pilot:{{< istio_full_version >}} istio=pilot
   {{< /text >}}

1. （オプション）Istio の入站ゲートウェイをインストールします：

   {{< text syntax=bash snip_id=install_ingressgateway >}}
   $ kubectl create namespace istio-ingress
   $ helm install istio-ingress istio/gateway -n istio-ingress --wait
   {{< /text >}}

   ゲートウェイのインストールについての詳細なドキュメントについては、[ゲートウェイのインストール](/ja/docs/setup/additional-setup/gateway/)を参照してください。

   {{< warning >}}
   ゲートウェイがデプロイされる名前空間には `istio-injection=disabled` ラベルを付けてはいけません。
   詳細については、[注入ポリシーの制御](/ja/docs/setup/additional-setup/sidecar-injection/#controlling-the-injection-policy)を参照してください。
   {{< /warning >}}

{{< tip >}}
Helm ポストレンダラーを使用して Helm Chart をカスタマイズする方法の詳細なドキュメントについては、
[高度な Helm Chart カスタマイズ](/ja/docs/setup/additional-setup/customize-installation-helm/)を参照してください。
{{< /tip >}}

## 更新 Istio 設定 {#updating-your-configuration}

独自のインストールパラメーターで前述の Istio Helm Chart のデフォルト動作をオーバーライドし、
Helm アップグレードフローに従って Istio メッシュシステムのカスタムインストールを行うことができます。
利用可能な設定オプションについては、`helm show values istio/<chart>` を使用して見つけることができます。
例：`helm show values istio/gateway`。

### 非 Helm インストールからの移行 {#migrating-from-non-helm-installations}

`istioctl` を使用してインストールされた Istio バージョン（Istio 1.5 以前）から Helm に移行する場合は、
現在の Istio コントロールプレーンリソースを削除し、上記の手順に従って Helm を使用して Istio を再インストールする必要があります。
現在の Istio を削除する際は、カスタム Istio リソースを失わないよう、Istio のカスタムリソース定義（CRD）を削除しないでください。

{{< warning >}}
推奨：クラスターから Istio を削除する前に、上記の手順を使用して Istio リソースをバックアップしてください。
{{< /warning >}}

[Istioctl アンインストールガイド](/ja/docs/setup/install/istioctl#uninstall-istio)で言及されている手順に従うことができます。

## アンインストール {#uninstall}

上記でインストールした Chart をアンインストールすることで、Istio とそのコンポーネントをアンインストールできます。

1. `istio-system` 名前空間にインストールされているすべての Istio Chart を一覧表示します：

   {{< text syntax=bash snip_id=helm_ls >}}
   $ helm ls -n istio-system
   NAME NAMESPACE REVISION UPDATED STATUS CHART APP VERSION
   istio-base istio-system 1 2024-04-17 22:14:45.964722028 +0000 UTC deployed base-{{< istio_full_version >}} {{< istio_full_version >}}
   istiod istio-system 1 2024-04-17 22:14:45.964722028 +0000 UTC deployed istiod-{{< istio_full_version >}} {{< istio_full_version >}}
   {{< /text >}}

1. （オプション）Istio のすべてのゲートウェイ Chart を削除します：

   {{< text syntax=bash snip_id=delete_delete_gateway_charts >}}
   $ helm delete istio-ingress -n istio-ingress
   $ kubectl delete namespace istio-ingress
   {{< /text >}}

1. Istio Discovery Chart を削除します：

   {{< text syntax=bash snip_id=helm_delete_discovery_chart >}}
   $ helm delete istiod -n istio-system
   {{< /text >}}

1. Istio Base Chart を削除します：

   {{< tip >}}
   設計上、Helm で Chart を削除しても、その Chart によってインストールされた CRD は削除されません。
   {{< /tip >}}

   {{< text syntax=bash snip_id=helm_delete_base_chart >}}
   $ helm delete istio-base -n istio-system
   {{< /text >}}

1. `istio-system` 名前空間を削除します：

   {{< text syntax=bash snip_id=delete_istio_system_namespace >}}
   $ kubectl delete namespace istio-system
   {{< /text >}}

## 安定したリビジョンラベルリソースのアンインストール {#uninstall-stable-revision-label-resources}

古いコントロールプレーンを引き続き使用する場合は、最初のリリースで新しいバージョンとそのラベルをアンインストールできます。
`helm template istiod istio/istiod -s templates/revision-tags.yaml --set revisionTags={prod-canary} --set revision=canary -n istio-system | kubectl delete -f -` を実行してください。
Istio のリビジョンをアンインストールするには、上記のアンインストール手順に従う必要があります。

このバージョンのゲートウェイをインプレースアップグレードでインストールした場合は、前のバージョンのゲートウェイを手動で再インストールする必要があります。
以前のバージョンとそのラベルを削除しても、以前のインプレースアップグレードされたゲートウェイは自動的に復元されません。

### （オプション）Istio がインストールした CRD の削除 {#deleting-customer-resource-definition-installed}

CRD を完全に削除すると、クラスターで作成したすべての Istio リソースが削除されます。
以下のコマンドでクラスターにインストールされた Istio CRD を完全に削除できます：

{{< text syntax=bash snip_id=delete_crds >}}
$ kubectl get crd -oname | grep --color=never 'istio.io' | xargs kubectl delete
{{< /text >}}

## インストール前にマニフェストを生成する {#generate-a-manifest-before-installation}

Istio をインストールする前に、各コンポーネントのマニフェストを生成するために `helm template` サブコマンドを使用できます。
例えば、`istiod` コンポーネントのマニフェストを生成するには、次のコマンドを使用します：

{{< text syntax=bash snip_id=none >}}
$ helm template istiod istio/istiod -n istio-system --kube-version {Kubernetes version of target cluster} > istiod.yaml
{{< /text >}}

生成されたマニフェストは、具体的に何がインストールされたかを確認し、マニフェストの変更を追跡するために使用できます。

{{< tip >}}
インストールに使用する通常のフラグまたはカスタム値も `helm template` コマンドに提供する必要があります。
{{< /tip >}}

上記で生成されたマニフェストを使用して、ターゲットクラスターで `istiod` コンポーネントをインストールできます：

{{< text syntax=bash snip_id=none >}}
$ kubectl apply -f istiod.yaml
{{< /text >}}

{{< warning >}}
`helm template` を使用して Istio をインストールおよび管理する場合は、以下の注意事項に注意してください：

1. Istio 名前空間（デフォルトでは `istio-system`）を手動で作成する必要があります。

1. リソースは、`helm install` と同じ依存関係の順序でインストールされない可能性があります。

1. この方法は、Istio バージョンの一部としてまだテストされていません。

1. `helm install` は、Kubernetes コンテキスト内の特定の環境設定を自動的に検出しますが、
   `helm template` はオフラインで実行されるため、これを実行できません。
   これは、予期しない結果を引き起こす可能性があります。特に、Kubernetes 環境がサードパーティのサービスアカウントトークンをサポートしていない場合は、
   あなたは、[これらの手順](/ja/docs/ops/best-practices/security/#configure-third-party-service-account-tokens)に従う必要があります。

1. クラスター内のリソースが正しい順序で利用できないため、生成されたマニフェストの `kubectl apply` で一時的なエラーが表示される可能性があります。

1. `helm install` は、設定の変更時に削除する必要があるリソースを自動的に削除します（例えば、ゲートウェイを削除した場合）。
   `helm template` と `kubectl` を使用する場合、これは発生しません。これらのリソースを手動で削除する必要があります。

{{< /warning >}}
