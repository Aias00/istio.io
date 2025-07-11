---
title: クイックスタート
description: シンプルな例でインストールを始める方法を紹介します。
weight: 50
keywords: [introduction]
owner: istio/wg-docs-maintainers-chinese
skip_seealso: true
test: n/a
---

Istio にご関心をお寄せいただきありがとうございます！

Istio には主に 2 つのモードがあります：**Ambient モード**と **Sidecar モード**。

- [Ambient モード](/ja/docs/overview/dataplane-modes/#ambient-mode)は新しい改良モデルで、
  Sidecar モードの課題を解決するために設計されています。Ambient モードでは各ノードにセキュアトンネルがインストールされ、
  必要に応じて（通常はネームスペース単位で）プロキシを追加して全機能を有効化できます。
- [Sidecar モード](/ja/docs/overview/dataplane-modes/#sidecar-mode)は
  2017 年に Istio が最初に導入した従来型サービスメッシュモデルです。Sidecar モードでは、
  各 Kubernetes Pod や他のワークロードごとにプロキシがデプロイされます。

Istio コミュニティは現在、主に Ambient モードの改良に注力していますが、
Sidecar モードも引き続き完全にサポートされています。新機能の多くは両モードで動作することが期待されています。

**新規ユーザーには Ambient モードから始めることを推奨します。**
Ambient モードは高速・低コストで管理も容易です。一部の[高度なユースケース](/ja/docs/overview/dataplane-modes/#unsupported-features)では Sidecar モードが必要ですが、
これらの課題も 2025 年のロードマップで解決を目指しています。

<div style="text-align: center;">
  <div style="display: inline-block;">
    <a href="/ja/docs/ambient/getting-started"
       style="display: inline-block; min-width: 18em; margin: 0.5em;"
       class="btn btn--secondary"
       id="get-started-ambient">Ambient モードで始める</a>
    <a href="/ja/docs/setup/getting-started"
       style="display: inline-block; min-width: 18em; margin: 0.5em;"
       class="btn btn--secondary"
       id="get-started-sidecar">Sidecar モードで始める</a>
  </div>
</div>
