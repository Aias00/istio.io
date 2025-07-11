---
title: 入門
description: Ambient モードで Istio をデプロイおよびインストールする方法。
weight: 2
aliases:
  - /zh/docs/ops/ambient/getting-started
  - /zh/latest/docs/ops/ambient/getting-started
owner: istio/wg-networking-maintainers
test: yes
skip_list: true
next: /docs/ambient/getting-started/deploy-sample-app
---

このガイドでは、Istio の {{< gloss "ambient" >}}Ambient モード{{< /gloss >}} を素早く評価できます。
続行するには Kubernetes クラスターが必要です。クラスターをお持ちでない場合は、
[kind](/ja/docs/setup/platform-setup/kind) またはその他の[サポートされている Kubernetes プラットフォーム](/ja/docs/setup/platform-setup/) を使用できます。

これらの手順では、Kubernetes の[サポートされているバージョン](/ja/docs/releases/supported-releases#support-status-of-istio-releases)
（{{< supported_kubernetes_versions >}}）が実行されている {{< gloss "cluster" >}}クラスター{{< /gloss >}} が必要です。

## Istio CLI のダウンロード {#download-the-istio-cli}

Istio は `istioctl` というコマンドラインツールで構成します。このツールと Istio のサンプルアプリケーションをダウンロードします：

{{< text syntax=bash snip_id=none >}}
$ curl -L https://istio.io/downloadIstio | sh -
$ cd istio-{{< istio_full_version >}}
$ export PATH=$PWD/bin:$PATH
{{< /text >}}

`istioctl` を実行できるかどうか、バージョンを表示するコマンドで確認します。
この時点では、Istio はまだクラスターにインストールされていないため、Pod が準備されていないことが表示されます。

{{< text syntax=bash snip_id=none >}}
$ istioctl version
Istio is not present in the cluster: no running Istio pods in namespace "istio-system"
client version: {{< istio_full_version >}}
{{< /text >}}

## Istio をクラスターにインストールする {#install-istio-on-to-your-cluster}

`istioctl` は複数の[プロファイル](/ja/docs/setup/additional-setup/config-profiles/)をサポートしており、
それぞれ異なるデフォルトオプションが含まれており、本番環境のニーズに合わせてカスタマイズできます。
`ambient` プロファイルには Ambient モードのサポートが含まれています。次のコマンドで Istio をインストールします：

{{< text syntax=bash snip_id=install_ambient >}}
$ istioctl install --set profile=ambient --skip-confirmation
{{< /text >}}

インストールが完了すると、すべてのコンポーネントが正常にインストールされたことを示す次の出力が表示されます。

{{< text syntax=plain snip_id=none >}}
✔ Istio core installed
✔ Istiod installed
✔ CNI installed
✔ Ztunnel installed
✔ Installation complete
{{< /text >}}

## Kubernetes Gateway API CRD のインストール {#install-the-kubernetes-gateway-api-crds}

Kubernetes Gateway API を使用してトラフィックルーティングを構成します。

{{< boilerplate gateway-api-install-crds >}}

## 次のステップ {#next-steps}

おめでとうございます！Ambient モード対応の Istio のインストールに成功しました。
次のステップに進み、[サンプルアプリケーションをインストール](/ja/docs/ambient/getting-started/deploy-sample-app/) してください。
