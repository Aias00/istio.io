---
title: minikube
description: minikube 上での Istio 設定。
weight: 50
skip_seealso: true
aliases:
  - /zh/docs/setup/kubernetes/prepare/platform-setup/minikube/
  - /zh/docs/setup/kubernetes/platform-setup/minikube/
keywords: [platform-setup, kubernetes, minikube]
owner: istio/wg-environments-maintainers
test: no
---

ドキュメントに従って minikube をインストールし、Istio や基本的なアプリケーションのために十分なシステムリソースを確保してください。

## 前提条件 {#prerequisites}

- minikube の実行には管理者権限が必要です。

- [シークレットディスカバリーサービス](https://www.envoyproxy.io/docs/envoy/latest/configuration/security/secret#sds-configuration)（SDS）を有効にする場合、
  Kubernetes Deployment に[追加設定](https://kubernetes.io/ja/docs/tasks/configure-pod-container/configure-service-account/#service-account-token-volume-projection)が必要です。
  最新のオプションパラメータについては [`api-server` リファレンス](https://kubernetes.io/ja/docs/reference/command-line-tools-reference/kube-apiserver/)を参照してください。

## インストール手順 {#installation-steps}

1. 最新版の [minikube](https://kubernetes.io/ja/docs/tasks/tools/#minikube)
   および [minikube 仮想マシンドライバ](https://minikube.sigs.k8s.io/docs/start/#install-a-hypervisor) をインストールします。

1. デフォルト以外のドライバを使用する場合は、minikube 仮想マシンドライバを設定してください。

   例えば、KVM 仮想マシンをインストールしている場合、次のコマンドで minikube の `driver` 設定を行います：

   {{< text bash >}}
   $ minikube config set driver kvm2
   {{< /text >}}

1. 16384 `MB` のメモリと 4 `CPUs` で minikube を起動します。この例では Kubernetes **1.26.1** を使用しています。
   `--kubernetes-version` の値を設定することで、任意の Istio 対応 Kubernetes バージョンを指定できます：

   {{< text bash >}}
   $ minikube start --memory=16384 --cpus=4 --kubernetes-version=v1.26.1
   {{< /text >}}

   使用する仮想マシンやプラットフォームによって最小メモリ要件は異なりますが、16384 `MB` で Istio と bookinfo の両方が十分に動作します。

   {{< tip >}}
   minikube 仮想マシンに十分なメモリが割り当てられていない場合、以下のようなエラーが発生することがあります：

   - イメージのプル失敗
   - ヘルスチェックのタイムアウト失敗
   - ホスト上での kubectl の失敗
   - 仮想マシンやホストのネットワーク不安定
   - 仮想マシンの完全なフリーズ
   - ホスト NMI ウォッチドッグによる再起動

   minikube にはメモリ使用量を確認する便利な方法があります。minikube 仮想マシンに `ssh` で接続し、コマンドラインで top コマンドを実行します：

   {{< text bash >}}
   $ minikube ssh
   {{< /text >}}

   {{< text bash >}}
   $ top
   GiB Mem : 12.4/15.7
   {{< /text >}}

   この例では、仮想マシン全体の 15.7G メモリのうち 12.4G が使用中です。これは 16G メモリの Macbook Pro 13" 上で、Istio 1.2 と bookinfo を VMWare Fusion 仮想マシンで動作させた際のデータです。
   {{< /tip >}}

1. （オプション・推奨）minikube で Istio 用のロードバランサを提供したい場合は、
   [minikube tunnel](https://minikube.sigs.k8s.io/docs/tasks/loadbalancer/#using-minikube-tunnel) を利用できます。
   minikube tunnel はネットワーク診断情報を表示するため、別のターミナルで実行してください：

   {{< text bash >}}
   $ minikube tunnel
   {{< /text >}}

   {{< warning >}}
   minikube が tunnel network を正しくクリーンアップしない場合があります。強制的にクリーンアップするには次のコマンドを使用してください：

   {{< text bash >}}
   $ minikube tunnel --cleanup
   {{< /text >}}

   {{< /warning >}}
