---
title: MicroK8s
description: Istio を利用するための MicroK8s の設定。
weight: 45
skip_seealso: true
aliases:
  - /zh/docs/setup/kubernetes/prepare/platform-setup/MicroK8s/
  - /zh/docs/setup/kubernetes/platform-setup/MicroK8s/
keywords: [platform-setup, kubernetes, MicroK8s]
owner: istio/wg-environments-maintainers
test: no
---

このページの最終英語更新日は 2019 年 8 月 28 日です。

{{< boilerplate untested-document >}}

以下の手順に従って、Istio を利用するために MicroK8s を設定してください。

{{< warning >}}
MicroK8s の実行には管理者権限が必要です。
{{< /warning >}}

1. 次のコマンドで最新バージョンの [MicroK8s](https://microk8s.io) をインストールします。

   {{< text bash >}}
   $ sudo snap install microk8s --classic
   {{< /text >}}

1. 次のコマンドで Istio を有効化します。

   {{< text bash >}}
   $ microk8s.enable istio
   {{< /text >}}

1. プロンプトが表示された場合、sidecar 間で双方向 TLS 認証を強制するかどうかを選択します。
   Istio 非対応サービスと対応サービスが混在している場合や、よく分からない場合は「No」を選択してください。

以下のコマンドでデプロイの進行状況を確認できます：

    {{< text bash >}}
    $ watch microk8s.kubectl get all --all-namespaces
    {{< /text >}}
