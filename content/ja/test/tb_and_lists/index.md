---
title: テキストブロックとリスト
description: テキストブロックとリストの作成。
skip_sitemap: true
---

1. 番号付きの項目

   {{< text plain >}}
   番号付きの項目にネストされたテキストブロック
   2 行目があります

   そして 3 行目
   {{< /text >}}

1. もう一つの番号付き項目

   {{< warning >}}
   ネストされた警告
   {{< /warning >}}

   {{< text plain >}}
   もう一つのネストされたテキストブロック
   2 行目があります

   そして 3 行目
   {{< /text >}}

1. さらに別の番号付き項目

   2 つ目の段落

1. さらに番号付きの項目

   {{< warning >}}
   これは番号付きの警告です。

   {{< text plain >}}
   これは番号付き警告内のテキストブロックです
   2 行目があります

   そして 3 行目
   {{< /text >}}

   {{< /warning >}}

1. `kubectl` コマンドでアプリケーションをデプロイします：

   {{< text bash >}}
   $ kubectl apply -f @samples/bookinfo/platform/kube/bookinfo.yaml@
   {{< /text >}}

   {{< warning >}}
   インストール時に自動サイドカーインジェクションを無効にし、
   [手動サイドカーインジェクション](/zh/docs/setup/additional-setup/sidecar-injection/#manual-sidecar-injection) を使用している場合は、
   まず `istioctl kube-inject` コマンドで `bookinfo.yaml` ファイルを修正してからアプリケーションをデプロイしてください。
   詳細は `istioctl` の [リファレンスドキュメント](/zh/docs/reference/commands/istioctl/#istioctl-kube-inject) を参照してください。

   {{< text bash >}}
   $ kubectl apply -f <(istioctl kube-inject -f @samples/bookinfo/platform/kube/bookinfo.yaml@)
   {{< /text >}}

   {{< /warning >}}

   このコマンドは `bookinfo` アプリケーションアーキテクチャ図に示されている 4 つのサービスすべてを起動します。
   レビューサービスの 3 つのバージョン（v1、v2、v3）もすべて起動されます。

   {{< tip >}}
   実際のデプロイでは、新しいバージョンのマイクロサービスは時間をかけてデプロイされ、すべてのバージョンが同時にデプロイされるわけではありません。
   {{< /tip >}}
