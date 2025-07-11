---
title: イメージ署名と検証
description: Istio イメージの出所をイメージ署名で検証する方法を説明します。
weight: 35
aliases: []
keywords: [install, signing]
owner: istio/wg-environments-maintainers
test: n/a
---

このページでは、[Cosign](https://github.com/sigstore/cosign) を使って Istio イメージアーティファクトの出所を検証する方法を紹介します。

Cosign は [sigstore](https://www.sigstore.dev) プロジェクトの一部として開発されたツールです。
署名済みの OCI（Open Container Initiative）アーティファクト（例：コンテナイメージ）の署名と検証を簡素化します。

Istio 1.12 以降、すべての公式リリースコンテナイメージはリリースプロセスの一部として署名されています。
最終ユーザーは、以下の手順でこれらのイメージを検証できます。

この手順は手動でも、ビルド／デプロイパイプラインと統合してイメージアーティファクトを自動検証する場合にも利用できます。

## 前提条件 {#prerequisites}

始める前に、以下を実施してください：

1. お使いのアーキテクチャ用の最新の [Cosign](https://github.com/sigstore/cosign/releases/latest) ビルドとその署名をダウンロードします。
1. `cosign` バイナリの署名を検証します：

   {{< text bash >}}
   $ openssl dgst -sha256 \
    -verify <(curl -ssL https://raw.githubusercontent.com/sigstore/cosign/main/release/release-cosign.pub) \
    -signature <(cat /path/to/cosign.sig | base64 -d) \
    /path/to/cosign-binary
   {{< /text >}}

1. バイナリを実行可能に（`chmod +x`）し、`PATH` 上のディレクトリに移動します。

## イメージの検証 {#validating-image}

コンテナイメージを検証するには、次の手順を実行します：

{{< text bash >}}
$ ./cosign-binary verify --key "https://istio.io/misc/istio-key.pub" {{< istio_docker_image "pilot" >}}
{{< /text >}}

この手順は、Istio のビルドインフラでビルドされたすべての公開済みまたは公開予定のイメージに適用できます。

出力例：

{{< text bash >}}
$ cosign verify --key "https://istio.io/misc/istio-key.pub" gcr.io/istio-release/pilot:1.12.0

gcr.io/istio-release/pilot:1.12.0 の検証——各署名について以下のチェックが行われました：

- 連名署名ステートメントが検証されました
- 署名が指定された公開鍵で検証されました
- すべての証明書が Fulcio ルートで検証されました

[{"critical":{"identity":{"docker-reference":"gcr.io/istio-release/pilot"},"image":{"docker-manifest-digest":"sha256:c37fd83f6435ca0966d653dc6ac42c9fe5ac11d0d5d719dfe97de84acbf7a32d"},"type":"cosign container image signature"},"optional":null}]
{{< /text >}}
