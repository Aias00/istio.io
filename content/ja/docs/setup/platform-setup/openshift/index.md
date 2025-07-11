---
title: OpenShift
description: OpenShift クラスタ上での Istio サービスのクイックセットアップ。
weight: 55
skip_seealso: true
aliases:
  - /zh/docs/setup/kubernetes/prepare/platform-setup/openshift/
  - /zh/docs/setup/kubernetes/platform-setup/openshift/
keywords: [platform-setup, openshift]
owner: istio/wg-environments-maintainers
test: no
---

以下の手順に従って、Istio 用の OpenShift クラスタを準備します。

OpenShift プロファイルを使って Istio をインストールします：

{{< text bash >}}
$ istioctl install --set profile=openshift
{{< /text >}}

Istio のインストールが完了したら、次のコマンドで Ingress Gateway 用の OpenShift ルートを公開します：

{{< text bash >}}
$ oc -n istio-system expose svc/istio-ingressgateway --port=http2
{{< /text >}}
