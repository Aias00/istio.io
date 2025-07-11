---
title: Docker Desktop
description: Docker Desktop で Istio を動作させるためのセットアップ手順。
weight: 15
skip_seealso: true
aliases:
  - /zh/docs/setup/kubernetes/prepare/platform-setup/docker-for-desktop/
  - /zh/docs/setup/kubernetes/prepare/platform-setup/docker/
  - /zh/docs/setup/kubernetes/platform-setup/docker/
keywords: [platform-setup, kubernetes, docker-desktop]
owner: istio/wg-environments-maintainers
test: no
---

1. Docker Desktop 上で Istio を動作させたい場合は、[サポートされている Kubernetes バージョン](/zh/docs/releases/supported-releases#support-status-of-istio-releases)
   ({{< supported_kubernetes_versions >}}) をインストールする必要があります。

1. Docker Desktop 内蔵の Kubernetes で Istio を動作させたい場合は、Docker Desktop の **Settings...** の
   **Resources->Advanced** パネルで Docker のメモリ制限を増やす必要があるかもしれません。リソースを最低でも 8.0 `GB` のメモリと 4 コアの `CPUs` に設定してください。

   {{< image width="60%" link="./dockerprefs.png"  caption="Docker の設定"  >}}

   {{< warning >}}
   最低限必要なメモリは環境によって異なります。8 `GB` あれば
   Istio と Bookinfo のインスタンスを動作させるのに十分です。Docker Desktop に十分なメモリが割り当てられていない場合、
   以下のようなエラーが発生する可能性があります：

   - イメージのプル失敗
   - ヘルスチェックのタイムアウト失敗
   - ホスト上での kubectl 実行失敗
   - 仮想マシンマネージャのネットワーク不安定

   以下のコマンドで Docker Desktop のリソースをさらに解放できます：

   {{< text bash >}}
   $ docker system prune
   {{< /text >}}

   {{< /warning >}}
