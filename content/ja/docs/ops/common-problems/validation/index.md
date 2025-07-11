---
title: 検証（バリデーション）に関する問題
description: 設定検証の問題を解決する方法。
force_inline_toc: true
weight: 50
aliases:
  - /zh/help/ops/setup/validation
  - /zh/help/ops/troubleshooting/validation
  - /zh/docs/ops/troubleshooting/validation
owner: istio/wg-user-experience-maintainers
test: no
---

## 一見有効な設定が反映されない {#valid-configuration-is-rejected}

[istioctl validate -f](/ja/docs/reference/commands/istioctl/#istioctl-validate) や [istioctl analyze](/ja/docs/reference/commands/istioctl/#istioctl-analyze) を使って、なぜ設定が反映されないのか追加情報を取得できます。コントロールプレーンのバージョンと近い **istioctl** CLI を使用してください。

最もよくある設定の問題は、YAML ファイルのスペースインデントや配列記号（`-`）のミスです。

設定が正しいか手動で確認し、必要に応じて [Istio API ドキュメント](/ja/docs/reference/config) を参照してください。

## 無効な設定が受け入れられる {#invalid-configuration-is-accepted}

`istio-validator-` で始まり、`<revision>-`（デフォルトでなければ）と Istio システムネームスペースが続く `validatingwebhookconfiguration` が正しく存在するか確認してください（例: `istio-validator-myrev-istio-system`）。
有効な設定の `apiVersion`、`apiGroup`、`resource` は `validatingwebhookconfiguration` の `webhooks` セクションに列挙されている必要があります。

{{< text bash yaml >}}
$ kubectl get validatingwebhookconfiguration istio-validator-istio-system -o yaml
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingWebhookConfiguration
metadata:
labels:
app: istiod
install.operator.istio.io/owning-resource-namespace: istio-system
istio: istiod
istio.io/rev: default
operator.istio.io/component: Pilot
operator.istio.io/managed: Reconcile
operator.istio.io/version: unknown
release: istio
name: istio-validator-istio-system
resourceVersion: "615569"
uid: 112fed62-93e7-41c9-8cb1-b2665f392dd7
webhooks:

- admissionReviewVersions:
  - v1beta1
  - v1
    clientConfig:
    # caBundle は空であってはなりません。webhook
    # サービスはインストールされたサービスアカウントのシークレットから ca-cert を
    # 1 秒ごとに定期的に（再）更新します。
    caBundle: LS0t...
    # service は webhook を実装する Kubernetes サービスを指します
    service:
    name: istiod
    namespace: istio-system
    path: /validate
    port: 443
    failurePolicy: Fail
    matchPolicy: Equivalent
    name: rev.validation.istio.io
    namespaceSelector: {}
    objectSelector:
    matchExpressions:
    - key: istio.io/rev
      operator: In
      values: - default
      rules:
  - apiGroups: - security.istio.io - networking.istio.io - telemetry.istio.io - extensions.istio.io
    apiVersions: - '_'
    operations: - CREATE - UPDATE
    resources: - '_'
    scope: '\*'
    sideEffects: None
    timeoutSeconds: 10
    {{< /text >}}

`istio-validator-` webhook が存在しない場合は、`global.configValidation` インストールオプションが `true` になっているか確認してください。

検証設定が失敗すると自動的に無効化されます。設定が存在し、スコープが正しければ webhook が呼び出されます。
リソースの作成や更新時に `caBundle` が空、証明書エラー、ネットワーク接続エラーがあるとエラーになります。
設定に問題がないのに webhook が呼ばれずエラーも出ない場合、クラスタの設定に問題がある可能性が高いです。

## 設定作成時のエラー：x509 証明書エラー {#x509-certificate-errors}

`x509: certificate signed by unknown authority` エラーは、webhook 設定の `caBundle` が空であることが原因である場合が多いです（[webhook 設定の検証](#invalid-configuration-is-accepted) を参照）。
Istio は `istio-validation` `configmap` とルート証明書を使い、webhook 設定を調整します。

1. `istiod` Pod が稼働しているか確認します：

   {{< text bash >}}
   $ kubectl -n istio-system get pod -lapp=istiod
   NAME READY STATUS RESTARTS AGE
   istiod-5dbbbdb746-d676g 1/1 Running 0 2d
   {{< /text >}}

1. Pod のログにエラーがないか確認します。`caBundle` の修正に失敗するとエラーが出ます：

   {{< text bash >}}
   $ for pod in $(kubectl -n istio-system get pod -lapp=istiod -o jsonpath='{.items[*].metadata.name}'); do \
    kubectl -n istio-system logs ${pod} \
   done
   {{< /text >}}

1. 修正に失敗した場合は、Istiod の RBAC 設定を確認します：

   {{< text bash yaml >}}
   $ kubectl get clusterrole istiod-istio-system -o yaml
   apiVersion: rbac.authorization.k8s.io/v1
   kind: ClusterRole
   name: istiod-istio-system
   rules:

   - apiGroups:
     - admissionregistration.k8s.io
       resources:
     - validatingwebhookconfigurations
       verbs:
     - '\*'
       {{< /text >}}

   Istio には `validatingwebhookconfigurations` への書き込み権限が必要です。

## 設定作成時のエラー：`no such hosts` または `no endpoints available` {#creating-configuration-fail}

検証が失敗すると自動的に無効化されます。`istiod` Pod が Ready でない場合、設定は作成・更新されません。下記の例のように `no endpoints available` エラーが出ることがあります。

`istiod` Pod が稼働しているか、エンドポイントが Ready か確認してください。

{{< text bash >}}
$ kubectl -n istio-system get pod -lapp=istiod
NAME READY STATUS RESTARTS AGE
istiod-5dbbbdb746-d676g 1/1 Running 0 2d
{{< /text >}}

{{< text bash >}}
$ kubectl -n istio-system get endpoints istiod
NAME ENDPOINTS AGE
istiod 10.48.6.108:15014,10.48.6.108:443 3d
{{< /text >}}

Pod やエンドポイントが Ready でない場合は、Pod のログや webhook Pod の起動を妨げている異常状態、サービスのトラフィックを確認してください。

{{< text bash >}}
$ for pod in $(kubectl -n istio-system get pod -lapp=istiod -o jsonpath='{.items[*].metadata.name}'); do \
 kubectl -n istio-system logs ${pod} \
 done
{{< /text >}}

{{< text bash >}}
$ for pod in $(kubectl -n istio-system get pod -lapp=istiod -o name); do \
 kubectl -n istio-system describe ${pod} \
 done
{{< /text >}}
