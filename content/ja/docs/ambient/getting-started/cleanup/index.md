---
title: クリーンアップ
description: Istio と関連リソースを削除します。
weight: 6
owner: istio/wg-networking-maintainers
test: yes
next: /docs/ambient/install
---

Istio と関連リソースが不要になった場合、本節の手順に従って削除できます。

## waypoint プロキシを削除 {#remove-waypoint-proxies}

すべての waypoint プロキシを削除するには、以下のコマンドを実行してください：

{{< text bash >}}
$ kubectl label namespace default istio.io/use-waypoint-
$ istioctl waypoint delete --all
{{< /text >}}

## Ambient データプレーンから名前空間を削除 {#remove-the-namespace-from-the-ambient-data-plane}

Istio を削除すると、`default` 名前空間のアプリケーションが Ambient メッシュに含まれるラベルは削除されません。
以下のコマンドを使用して削除します：

{{< text bash >}}
$ kubectl label namespace default istio.io/dataplane-mode-
{{< /text >}}

Istio をアンインストールする前に、Ambient データプレーンからワークロードを削除する必要があります。

## サンプルアプリケーションを削除 {#remove-the-sample-application}

Bookinfo サンプルアプリケーションと `curl` デプロイメントを削除するには、以下のコマンドを実行してください：

{{< text bash >}}
$ kubectl delete httproute reviews
$ kubectl delete authorizationpolicy productpage-viewer
$ kubectl delete -f @samples/curl/curl.yaml@
$ kubectl delete -f @samples/bookinfo/platform/kube/bookinfo.yaml@
$ kubectl delete -f @samples/bookinfo/platform/kube/bookinfo-versions.yaml@
$ kubectl delete -f @samples/bookinfo/gateway-api/bookinfo-gateway.yaml@

{{< /text >}}

## Istio をアンインストール {#uninstall-istio}

Istio をアンインストールするには：

{{< text syntax=bash snip_id=none >}}
$ istioctl uninstall -y --purge
$ kubectl delete namespace istio-system
{{< /text >}}

## Kubernetes Gateway API CRD を削除 {#remove-the-kubernetes-gateway-api-crds}

{{< boilerplate gateway-api-remove-crds >}}
