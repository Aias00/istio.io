---
title: インプレースアップグレード
description: インプレースアップグレードとロールバック。
weight: 20
keywords: [kubernetes, upgrading, in-place]
owner: istio/wg-environments-maintainers
test: no
---

`istioctl upgrade` コマンドで Istio をアップグレードします。

{{< tip >}}
[カナリアアップグレード](/ja/docs/setup/upgrade/canary/)はインプレースアップグレードよりも安全で、推奨されるアップグレード方法です。
{{< /tip >}}

Istio のアップグレードコマンドはロールバック操作にも利用できます。

[`istioctl` アップグレードリファレンス](/ja/docs/reference/commands/istioctl/#istioctl-upgrade)で
`istioctl upgrade` コマンドの全パラメータを確認できます。

{{< warning >}}
`istioctl upgrade` はインプレースアップグレード用であり、`--revision` パラメータを使ったインストールとは互換性がありません。
このようなインストールのアップグレードは失敗し、エラーが表示されます。
{{< /warning >}}

## アップグレードの前提条件 {#upgrade-prerequisites}

アップグレードを実行する前に、以下の条件を確認してください：

- インストール済みの Istio バージョンとアップグレード先バージョンの差は最大で 1 つのマイナーバージョンであること。
  例：1.7.x へアップグレードする場合は、少なくとも 1.6.0 以上である必要があります。

- [{{< istioctl >}} でインストール](/ja/docs/setup/install/istioctl/)した Istio であること。

## アップグレード手順 {#upgrade-steps}

{{< warning >}}
アップグレード中はサービスのトラフィックが一時的に中断される場合があります。中断を最小限に抑えるため、`istiod` のレプリカが少なくとも 2 つ稼働していることを確認してください。
また、[`PodDisruptionBudgets`](https://kubernetes.io/ja/docs/tasks/run-application/configure-pdb/) の最小可用性が 1 であることも確認してください。
{{< /warning >}}

このセクションのすべてのコマンドは新バージョンの `istioctl` で実行してください。実行ファイルはダウンロードパッケージの `bin/` ディレクトリにあります。

1. [新しい Istio をダウンロード](/ja/docs/setup/additional-setup/download-istio-release/)し、そのディレクトリに移動します。

1. Kubernetes 設定がアップグレード対象クラスタを指していることを確認します：

   {{< text bash >}}
   $ kubectl config view
   {{< /text >}}

1. アップグレードが環境と互換性があるか確認します：

   {{< text bash >}}
   $ istioctl x precheck
   ✔ No issues found when checking the cluster. Istio is safe to install or upgrade!
   To get started, check out https://istio.io/latest/docs/setup/getting-started/
   {{< /text >}}

1. 次のコマンドでアップグレードを開始します：

   {{< text bash >}}
   $ istioctl upgrade
   {{< /text >}}

   {{< warning >}}
   `-f` パラメータで Istio をインストールした場合は、`istioctl upgrade` 実行時にも同じ `-f` パラメータ値を指定してください。
   例：`istioctl install -f <IstioOperator-custom-resource-definition-file>`。
   {{< /warning >}}

   `--set` パラメータでインストールした場合も、アップグレード時に同じ `--set` パラメータ値を指定してください。
   そうしないとアップグレード時に `--set` の値がリセットされます。運用環境では `--set` ではなく設定ファイルでのインストールを推奨します。

   `-f` パラメータを指定しない場合、Istio はデフォルトプロファイルでアップグレードされます。

   いくつかのチェックの後、`istioctl` はアップグレードを続行するかどうかを確認します。

1. `istioctl` は Istio のコントロールプレーンとゲートウェイを新バージョンにアップグレードし、完了ステータスを表示します。

1. `istioctl` でアップグレードが完了したら、Pod の Sidecar を再起動して Istio のデータプレーンを手動で更新してください。

   {{< text bash >}}
   $ kubectl rollout restart deployment
   {{< /text >}}

## バージョンロールバックの前提条件 {#downgrade-prerequisites}

ロールバックを開始する前に、以下の前提条件を確認してください：

- [{{< istioctl >}} でインストール](/ja/docs/setup/install/istioctl/)した Istio であること。

- インストール済みの Istio バージョンとロールバック先バージョンの差は最大で 1 つのマイナーバージョンであること。
  例：1.7.x から最小 1.6.0 までロールバック可能です。

- ロールバックはロールバック先バージョンの `istioctl` バイナリで実行する必要があります。
  例：Istio 1.7 から 1.6.5 へロールバックする場合は 1.6.5 の `istioctl` を使用してください。

## ロールバック手順 {#steps-to-downgrade-to-a-lower-Istio-version}

`istioctl upgrade` を使って Istio を低いバージョンにロールバックできます。手順は前述のアップグレードと同じですが、
低いバージョン（例：1.6.5）の `istioctl` バイナリを使って実行します。完了後、Istio は低いバージョンに更新されます。

また、`istioctl install` で旧バージョンの Istio コントロールプレーンをインストールすることもできます。
