---
title: 動的アドミッション Webhook の概要
description: Istio における Kubernetes webhook の利用と関連する問題について簡単に説明します。
weight: 10
aliases:
  - /zh/help/ops/setup/webhook
  - /zh/docs/ops/setup/webhook
owner: istio/wg-user-experience-maintainers
test: no
---

[Kubernetes のアドミッションコントロールメカニズム](https://kubernetes.io/ja/docs/reference/access-authn-authz/extensible-admission-controllers/)より：

{{< tip >}}
アドミッション Webhook は HTTP で呼び出されるコールバックで、アドミッションリクエストを受け取り処理します。
アドミッション Webhook には 2 種類あり、Validating アドミッション Webhook と Mutating アドミッション Webhook です。
Validating Webhook ではカスタムのアドミッションポリシーでリクエストを拒否でき、Mutating Webhook ではカスタムのデフォルト値でリクエストを変更できます。
{{< /tip >}}

Istio は `ValidatingAdmissionWebhooks` を使って Istio 設定の検証を行い、
`MutatingAdmissionWebhooks` を使ってユーザー Pod への Sidecar プロキシ自動注入を行います。

Webhook の設定には Kubernetes の動的アドミッション Webhook の知識が必要です。
Kubernetes API の詳細は
[Mutating Webhook Configuration](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.29/#mutatingwebhookconfiguration-v1-admissionregistration-k8s-io)
および [Validating Webhook Configuration](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.29/#validatingwebhookconfiguration-v1-admissionregistration-k8s-io) を参照してください。

## 動的アドミッション Webhook の前提条件の確認 {#verify-dynamic-admission-webhook-prerequisites}

Kubernetes の[プラットフォームセットアップガイド](/ja/docs/setup/platform-setup/)を参照し、
Kubernetes の詳細なセットアップ手順を確認してください。クラスタ設定に問題があると Webhook は正常に動作しません。クラスタ設定後、
動的 Webhook や関連機能が正しく動作しない場合は、以下の手順で確認できます。

1. 現在利用している [`kubectl`](https://kubernetes.io/ja/docs/tasks/tools/#kubectl) と Kubernetes サーバーが[サポートされているバージョン](/ja/docs/releases/supported-releases#support-status-of-istio-releases) ({{< supported_kubernetes_versions >}}) であることを確認します：

   {{< text bash >}}
   $ kubectl version --short
   Client Version: v1.29.0
   Server Version: v1.29.1
   {{< /text >}}

1. `admissionregistration.k8s.io/v1beta1` が有効であることを確認します

   {{< text bash >}}
   $ kubectl api-versions | grep admissionregistration.k8s.io/v1
   admissionregistration.k8s.io/v1
   admissionregistration.k8s.io/v1beta1
   {{< /text >}}

1. `kube-apiserver --enable-admission-plugins` 設定で
   `MutatingAdmissionWebhook` および `ValidatingAdmissionWebhook` プラグインが有効になっていることを確認します。
   [指定仕様](/ja/docs/setup/platform-setup/)のフラグ（`--enable-admission-plugins`）を確認してください。

1. Kubernetes API Server と Webhook が動作する Pod 間のネットワーク接続が正常であることを確認します。
   例えば、`http_proxy` の誤設定は API Server の正常動作を妨げる場合があります（詳細は
   [PR](https://github.com/kubernetes/kubernetes/pull/58698#discussion_r163879443)
   および [Issue](https://github.com/kubernetes/kubeadm/issues/666) を参照）。
