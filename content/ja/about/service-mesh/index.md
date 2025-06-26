---
title: Istio サービス メッシュ
description: サービス メッシュ。
subtitle: Istio は、開発者とオペレーターが分散型またはマイクロサービス アーキテクチャで直面する課題を解決します。Istio は、新規から既存のアプリケーションをクラウドネイティブに移行する場合、または既存の資産を保護する場合にも役立ちます。
weight: 34
skip_toc: true
skip_byline: true
skip_pagenav: true
aliases:
    - /ja/service-mesh.html
    - /ja/docs/concepts/what-is-istio/overview
    - /ja/docs/concepts/what-is-istio/goals
    - /ja/about/intro
    - /ja/docs/concepts/what-is-istio/
    - /ja/latest/docs/concepts/what-is-istio/
doc_type: about
---

{{< centered_block >}}
{{< figure src="/ja/about/service-mesh/service-mesh.svg" alt="サービス メッシュ" title="アプリケーション プロキシを使用することで、Istio を使用して、ネットワーク内のアプリケーション感知型のトラフィック管理、驚くべき可観測性、および強力なセキュリティ機能をプログラムできます。" >}}
{{< /centered_block >}}

{{< centered_block >}}

[comment]: <> (この下のタイトルは、lint が最初のタイトルを <h2> と要求するため、後で <h1> が必要です。)

## Istio の紹介 {#what-is-Istio}

**サービス メッシュ**は、アプリケーションに対して、コードを変更することなく、ゼロトラスト セキュリティ、可観測性、および高度なトラフィック管理機能を提供するインフラストラクチャ レイヤーです。
**Istio** は、最も人気のある、最も強力な、最も信頼できるサービス メッシュです。
Istio は、Google、IBM、Lyft によって 2016 年に設立され、クラウドネイティブ コンピューティング ファウンデーションの卒業プロジェクトで、Kubernetes や Prometheus などのプロジェクトと並んでいます。

Istio は、クラウドネイティブ システムと分散型システムを弾力性を持たせ、現代企業が異なるプラットフォームをまたいで、接続を維持し、資産を保護しながら、ワークロードを維持するのに役立ちます。
[セキュリティとガバナンスの制御を有効にする](/ja/docs/concepts/observability/)、mTLS 暗号化、ポリシー管理、アクセス制御、
[ネットワーク機能をサポートする](/ja/docs/concepts/traffic-management/)、例えば、金絲雀デプロイ、A/B テスト、負荷分散、障害復旧、
および[資産全体のトラフィックの可観測性を増やす](/ja/docs/concepts/observability/)。

Istio は、単一のクラスター、ネットワーク、または実行時の境界に限定されません。Kubernetes または VM、クラウド、ハイブリッド、またはオンプレミスで実行されるサービスは、単一のメッシュに含まれる場合があります。

Istio は、拡張性があり、貢献者とパートナーの[広範なエコシステム](/ja/about/ecosystem)によってサポートされています。
Istio は、さまざまなユースケースに対して、パッケージ化された統合と配布を提供します。Istio は、独立してインストールすることも、Istio ベースのソリューションを提供する商用サプライヤーによるホスティング サポートを選択することもできます。

<div class="cta-container">
    <a class="btn" href="/ja/docs/overview/">Istio についてもっと詳しく知る</a>
</div>

{{< /centered_block >}}

<br/><br/>

# 機能 {#features}

{{< feature_block header="デフォルトのセキュリティ" image="security.svg" >}}
Istio は、ワークロード ID、双方向 TLS、および強力なポリシー制御に基づく、市場で最も優れたゼロトラスト ソリューションを提供します。
Istio は、[BeyondProd](https://cloud.google.com/security/beyondprod/) の価値をオープンソースで実現し、サプライヤーのロックインや SPOF を回避します。

<a class="btn" href="/ja/docs/concepts/security/">セキュリティについて詳しく知る</a>
{{< /feature_block>}}

{{< feature_block header="可観測性の向上" image="observability.svg" >}}
Istio は、サービス メッシュ内で可観測データを生成し、サービスの動作を可観測性にするのに役立ちます。
Grafana や Prometheus などの APM システムと統合され、運用者に洞察的な指標を提供し、障害の排除、メンテナンス、およびアプリケ
{{< /feature_block>}}

{{< feature_block header="トラフィック管理" image="management.svg" >}}
Istio は、トラフィック ルーティングとサービス レベルの設定を簡素化し、A/B テスト、金絲雀デプロイ、およびパーセンテージ トラフィック分割に基づく段階的なデプロイメントなどのタスクを容易に制御できます。

<a class="btn" href="/ja/docs/concepts/traffic-management/">トラフィック管理について詳しく知る</a>
{{< /feature_block>}}

<br/><br/>

# なぜ Istio を選択するのですか？ {#why-istio}

{{< feature_block header="複数のデプロイ モード" image="deployment-modes.svg" >}}
Istio は、ユーザーが選択できる 2 つのデータ プレーン モードを提供します。新しい Ambient モードを使用してアプリケーションのライフサイクルを簡素化するか、従来の Sidecar を使用して複雑な設定を行います。

<a class="btn" href="/ja/docs/overview/dataplane-modes/">データ プレーン モードについて詳しく知る</a>
{{< /feature_block>}}

{{< feature_block header="Envoy によるサポート" image="envoy.svg" >}}
Istio は、サービス メッシュの業界標準のゲートウェイ プロキシである Envoy に基づいて構築されており、高性能と拡張性を備えています。
WebAssembly を使用してカスタム トラフィック機能を追加するか、またはサードパーティのポリシー システムを統合します。

<a class="btn" href="/ja/docs/overview/why-choose-istio/#envoy">Istio と Envoy について詳しく知る</a>
{{< /feature_block>}}

{{< feature_block header="本当のコミュニティ プロジェクト" image="community-project.svg" >}}
Istio は、現代のワークロードを設計し、クラウドネイティブ 分野の巨大な革新者コミュニティによって構築されています。

<a class="btn" href="/ja/docs/overview/why-choose-istio/#community">Istio の貢献者について詳しく知る</a>
{{< /feature_block>}}

{{< feature_block header="安定したバイナリ バージョン" image="stable-releases.svg" >}}
自信に生産環境のワークロードに Istio をデプロイします。すべてのバージョンは完全に無料で使用できます。

<a class="btn" href="/ja/docs/overview/why-choose-istio/#packages">Istio のパッケージング方式について詳しく知る</a>
{{< /feature_block>}}
