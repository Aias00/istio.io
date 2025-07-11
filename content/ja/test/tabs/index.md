---
title: タブ
description: 基本的なタブ。
skip_sitemap: true
---

{{< tabset category-name="test" >}}

{{< tab name="One" category-value="one" >}}
1 つの段落
{{< /tab >}}

{{< tab name="Two" category-value="two" >}}
3 つの

別々の

段落
{{< /tab >}}

{{< tab name="Three" category-value="three" >}}
{{< warning >}}
タブ内の警告
{{< /warning >}}
{{< /tab >}}

{{< tab name="Four" category-value="four" >}}
シンプルなテキスト

2 つの段落

{{< warning >}}
タブ内の警告
{{< /warning >}}
{{< /tab >}}

{{< tab name="Five" category-value="five" >}}
シンプルなテキスト

{{< text plain >}}
タブ内のテキストブロック
{{< /text >}}

{{< /tab >}}

{{< tab name="Six" category-value="six" >}}
タブ内の _マークダウン_ を含むシンプルなテキスト

{{< warning >}}
タブ内の _マークダウン_ を含む警告

{{< text plain >}}
タブ内の警告内のテキストブロック
{{< /text >}}

さらに _マークダウン_
{{< /warning >}}

{{< /tab >}}

{{< tab name="Seven" category-value="seven" >}}
タブ内の _マークダウン_ を含むシンプルなテキスト

{{< text plain >}}
インデントなし：
4 つインデント： - 8 つインデント
再び 4 つインデント： - 再び 8 つインデント
{{< /text >}}

さらに _マークダウン_
{{< /tab >}}

{{< /tabset >}}
