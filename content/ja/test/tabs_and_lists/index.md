---
title: タブとリスト
description: タブとリストの組み合わせ。
skip_sitemap: true
---

{{< tabset category-name="test" >}}

{{< tab name="One" category-value="one" >}}

1. タブ内のリストの 1 つの段落
   {{< /tab >}}

{{< tab name="Two" category-value="two" >}}

1.  3 つの

1.  別々の

1.  タブ内のリストの箇条書き

        最後の箇条書きは2つの段落があります

    {{< /tab >}}

{{< tab name="Three" category-value="three" >}}

1. タブ内のリストのシンプルなテキスト

   段落

   {{< warning >}}
   タブ内のリストの警告
   {{< /warning >}}

   そしてもう一つ

1. 2 つ目の箇条書き

1. 3 つ目の箇条書き
   {{< /tab >}}

{{< tab name="Four" category-value="four" >}}

1.  タブ内のリストの _マークダウン_ を含むシンプルなテキスト

        {{< warning >}}
        タブ内のリストの警告
        {{< /warning >}}

    {{< /tab >}}

{{< tab name="Five" category-value="five" >}}

1. タブ内のリストのシンプルなテキスト

   {{< text plain >}}
   タブ内のリストのテキストブロック
   {{< /text >}}

{{< /tab >}}

{{< tab name="Six" category-value="six" >}}

1. タブ内のリストの _マークダウン_ を含むシンプルなテキスト

   {{< warning >}}
   タブ内のリストの _マークダウン_ を含む警告
   {{< /warning >}}

1. 2 つ目の箇条書き
   {{< /tab >}}

{{< tab name="Seven" category-value="seven" >}}

1. タブ内のリストの _マークダウン_ を含むシンプルなテキスト

   {{< text bash >}}
   $ インデントなし：
   4 つインデント： - 8 つインデント
   再び 4 つインデント： - 再び 8 つインデント
   {{< /text >}}

1. 2 つ目の箇条書き
   {{< /tab >}}

{{< /tabset >}}
