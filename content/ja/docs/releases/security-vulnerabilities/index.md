---
title: セキュリティ脆弱性
description: 我们如何处理安全漏洞。
weight: 35
aliases:
  - /zh/about/security-vulnerabilities
  - /zh/latest/about/security-vulnerabilities
owner: istio/wg-docs-maintainers
test: n/a
---

Istio のセキュリティ脆弱性を報告してくださるセキュリティ研究者やユーザーの皆様に感謝します。
私たちはすべての報告を徹底的に分析・評価します。

## 脆弱性の報告 {#reporting-a-vulnerability}

脆弱性を報告するには、詳細を記載したメールを
[istio-security-vulnerability-reports@googlegroups.com](mailto:istio-security-vulnerability-reports@googlegroups.com)
までお送りください。潜在的なセキュリティ脆弱性に関係しない通常のバグについては、
[バグ報告](/ja/docs/releases/bugs/)ページをご覧ください。

### いつセキュリティ脆弱性を報告すべきか {#when-to-report-a-security-vulnerability}

以下の場合はご報告ください：

- Istio に潜在的なセキュリティ脆弱性があると思われる場合
- 脆弱性が Istio にどのように影響するか不明な場合
- Istio が依存する他のプロジェクト（例：Envoy、Docker、Kubernetes）に脆弱性があると思われる場合

迷った場合は、非公開でご報告ください。これには以下が含まれますが、これらに限りません：

- いかなるクラッシュ、特に Envoy におけるもの
- セキュリティポリシー（認証や認可など）の回避や脆弱性
- 潜在的なサービス拒否（DoS）

### いつセキュリティ脆弱性を報告しないか {#when-not-to-report-a-security-vulnerability}

以下の場合は脆弱性報告を送らないでください：

- Istio コンポーネントのセキュリティ調整の支援が必要な場合
- セキュリティアップデートに関する支援が必要な場合
- 問題がセキュリティに関係しない場合
- ベースイメージの依存関係に関する問題（[ベースイメージ](#base-images)を参照）

## 評価 {#evaluation}

Istio セキュリティチームは、すべての脆弱性報告を 3 営業日以内に確認・分析します。

Istio セキュリティチームと共有された脆弱性情報は Istio プロジェクトに帰属し、
問題解決に必要な範囲でのみ共有され、他プロジェクトには伝達されません。

`triaged` から `identified fix`、`release planning` まで、
セキュリティ問題の進捗状況は随時フィードバックします。

## 問題の修正 {#fixing-the-issue}

セキュリティ脆弱性が十分に記述されたら、Istio チームが修正プログラムを開発します。
修正プログラムの開発・テストは、脆弱性情報の早期漏洩を防ぐため、プライベートな GitHub リポジトリで行われます。

## 早期開示 {#early-disclosure}

Istio プロジェクトは、セキュリティ脆弱性を非公開で早期に開示するためのメーリングリストを運用しています。
このリストは Istio と密接に連携するパートナー向けに、実用的な情報を提供するためのものです。
個人がセキュリティ問題を知るためのものではありません。

詳細は[早期開示のセキュリティ脆弱性](https://github.com/istio/community/blob/master/EARLY-DISCLOSURE.md)をご覧ください。

## 公開開示 {#public-disclosure}

公開開示を選択した当日、以下の一連の作業ができるだけ迅速に行われます：

- プライベート GitHub リポジトリの修正ブランチをパブリックリポジトリの該当ブランチにマージ
- リリースエンジニアが必要なバイナリを迅速にビルド・公開
- バイナリが利用可能になったら、以下のチャネルで告知：
  - [Istio ブログ](/ja/blog)
  - discuss.istio.io の [Announcements](https://discuss.istio.io/c/announcements) セクション
  - [Istio Twitter feed](https://twitter.com/IstioMesh)
  - Slack の [#announcements](https://istio.slack.com/messages/CFXS256EQ/) チャンネル

告知には、修正版にアップグレードする前に取れる緩和策があればできる限り記載します。
告知の推奨時刻は UTC の月曜～木曜 16:00 です。
これは太平洋時間の朝、ヨーロッパとアジアの夕方にあたります。

## ベースイメージ {#base-images}

Istio は `ubuntu` ベースと `distroless` ベースの 2 種類のデフォルト Docker イメージを提供しています。
詳細は[Docker コンテナイメージの強化](/ja/docs/ops/configuration/security/harden-docker-images/)をご覧ください。
これらのイメージには新たに発見された CVE 脆弱性が含まれる場合があります。
Istio セキュリティチームはこれらのイメージを自動スキャンし、既知の CVE 脆弱性が含まれていないことを確認しています。

イメージ内で CVE 脆弱性が検出された場合、新しいイメージが自動的にビルドされ、以降のすべてのビルドで使用されます。
また、セキュリティチームはこれらの脆弱性が Istio で直接悪用可能かどうかを分析します。
多くの場合、脆弱性はベースイメージのパッケージに存在しても、Istio で利用される際に悪用できません。
このような場合、新バージョンは CVE のみを理由にリリースされず、修正は次の定期リリースに含まれます。

したがって、ベースイメージの CVE 脆弱性は[報告](#reporting-a-vulnerability)すべきではありません。
Istio で悪用可能である証拠がある場合を除きます。

ベースイメージの CVE 脆弱性を減らしたい場合は、
[`distroless`](/ja/docs/ops/configuration/security/harden-docker-images/) ベースイメージの利用を強く推奨します。
