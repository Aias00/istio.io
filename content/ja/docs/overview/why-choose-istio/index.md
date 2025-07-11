---
title: なぜ Istio を選ぶのか？
description: Istio を他のサービスメッシュと比較します。
weight: 20
keywords: [comparison]
owner: istio/wg-docs-maintainers-english
test: n/a
---

Istio は 2017 年に Sidecar ベースのサービスメッシュという概念を最初に提唱しました。
プロジェクトは当初から、ゼロトラストネットワークのための標準ベースの双方向 TLS、
インテリジェントなトラフィックルーティング、メトリクス・ログ・トレースによる可観測性など、
サービスメッシュを定義する機能を備えていました。

その後も、
[マルチクラスタ・マルチネットワークトポロジ](/ja/docs/ops/deployment/deployment-models/)、
[WebAssembly 拡張性](/ja/docs/concepts/wasm/)、
[Kubernetes Gateway API の開発](/ja/blog/2022/gateway-api-beta/)、
[Ambient モード](/ja/docs/ambient/overview/)によるアプリ開発者の負担軽減など、
メッシュ分野の進化をリードしてきました。

Istio をサービスメッシュとして選ぶべき理由をいくつか紹介します。

## シンプルかつ強力 {#simple-and-powerful}

Kubernetes には数百の機能と多数の API がありますが、Istio も同様に、
シンプルなコマンドで始められ、必要に応じて強力な機能を段階的に利用できます。
他の「シンプル」なサービスメッシュは、Istio が初日から持っていた機能に追いつくのに何年もかかりました。

「備えあれば憂いなし」。必要なときに機能がないより、今は不要でも備えておく方が良いのです。

## Envoy プロキシ {#envoy}

Istio は当初から {{< gloss >}}Envoy{{< /gloss >}} プロキシを採用しています。
Envoy は Lyft によって開発された高性能サービスプロキシで、Istio は最初に Envoy を採用したプロジェクトです。
[Istio チームは最初の外部コントリビューター](https://eng.lyft.com/envoy-7-months-later-41986c2fd443)でもあります。
Envoy はその後、[Google Cloud のロードバランサ]や他の多くのサービスメッシュの基盤となりました。

Istio は Envoy のすべての機能と柔軟性を活用し、
[Istio チームが Envoy に実装した](/ja/blog/2020/wasm-announce/)世界最高水準の拡張性も備えています。

## コミュニティ {#community}

Istio は真のコミュニティプロジェクトです。2023 年には 10 社以上が 1,000 件超の貢献を行い、
いずれの企業も 25% を超える貢献はありません。
（[統計はこちら](https://istio.devstats.cncf.io/d/5/companies-table?var-period_name=Last%20year&var-metric=contributions&orgId=1)）。

これほど幅広い業界の支持を得ているサービスメッシュは他にありません。

## パッケージ {#packages}

Istio はすべてのリリースで安定したバイナリを提供し、
[最新および一部旧バージョン](/ja/docs/releases/supported-releases/)向けに無料のセキュリティパッチも定期的に公開しています。
多くのベンダーがより古いバージョンをサポートしていますが、
安定したオープンソースプロジェクトでは、ベンダーとの契約がセキュリティ確保の必須条件であってはなりません。

## 検討された代替案 {#alternatives-considered}

良い設計ドキュメントには、検討されたが採用されなかった代替案も含まれるべきです。

### なぜ「eBPF を使わない」のか？ {#why-not-use-ebpf}

Istio も適切な場面では {{< gloss >}}eBPF{{< /gloss >}} を利用します。
[Merbridge](/ja/blog/2022/merbridge/) で Pod からプロキシへのトラフィック転送に eBPF を使うことができます。
`iptables` よりもわずかにパフォーマンスが向上します。

なぜすべての機能を eBPF で実装しないのか？
eBPF は Linux カーネル内で動作する仮想マシンで、
限られた計算リソースで完了することが保証された機能のみを実装できます。
L3 ルーティングや可観測性などは得意ですが、
Envoy のような複雑な長時間処理には向いていません。

他の「eBPF サービスメッシュ」も、実際にはノードごとの Envoy などユーザー空間ツールを併用しています。

### なぜノードごとのプロキシを使わないのか？ {#why-not-use-a-per-node-proxy}

Envoy はマルチテナント設計ではありません。
複数テナントの L7 トラフィックを 1 つの共有インスタンスで処理するのは、
セキュリティや安定性の観点から大きな課題があります。
Kubernetes では任意の名前空間の Pod が任意のノードにスケジューリングされるため、
ノードは適切なテナント境界ではありません。
L7 処理のコストも高く、コスト配分も困難です。

Ambient モードでは、ztunnel で L4 のみを処理し、
L7 はネームスペース単位の Envoy で行うことで、
安全かつ効率的な構成を実現しています。

## CNI だけで十分？ {#i-have-a-cni-why-do-i-need-istio}

最近は一部の CNI プラグインがサービスメッシュ的な機能を追加していますが、
これらは独自実装であり、特定の CNI 上でしか動作しません。
また、機能やセキュリティ要件も異なります。

Istio は業界標準の暗号化プロトコルを使った ztunnel で、
どの CNI やクラウドでも一貫したサービスメッシュを実現します。
[ztunnel の詳細はこちら](/ja/docs/ambient/overview)。

Istio は[業界標準の mTLS](/ja/docs/concepts/security/#mutual-tls-authetication)や
[強力な L7 ポリシー](/ja/docs/concepts/security/#authorization)、
[プラットフォーム非依存のワークロード ID](/ja/docs/concepts/security/#istio-identity)を提供します。
