---
title: トラフィック管理
description: Istio の多様なトラフィックルーティングと制御機能について説明します。
weight: 20
keywords:
  [traffic-management, pilot, envoy-proxies, service-discovery, load-balancing]
aliases:
  - /zh/docs/concepts/traffic-management/pilot
  - /zh/docs/concepts/traffic-management/rules-configuration
  - /zh/docs/concepts/traffic-management/fault-injection
  - /zh/docs/concepts/traffic-management/handling-failures
  - /zh/docs/concepts/traffic-management/load-balancing
  - /zh/docs/concepts/traffic-management/request-routing
  - /zh/docs/concepts/traffic-management/pilot.html
  - /zh/docs/concepts/traffic-management/overview.html
owner: istio/wg-networking-maintainers
test: n/a
---

Istio のトラフィックルーティングルールを使うと、サービス間のトラフィックや API コールを簡単に制御できます。Istio はサービスレベルの属性（サーキットブレーカー、タイムアウト、リトライなど）の設定を簡素化し、A/B テスト、カナリアリリース、トラフィックのパーセンテージ分割による段階的リリースなどの重要なタスクも容易に設定できます。また、堅牢性を高めるための障害復旧機能も標準で備えており、依存サービスやネットワーク障害時にもアプリケーションの健全性を維持できます。

Istio のトラフィック管理モデルは、サービスとともにデプロイされる {{< gloss >}}Envoy{{</ gloss >}} プロキシに基づいています。メッシュ内のサービスが送受信するすべての {{< gloss >}}データプレーン{{</ gloss >}} トラフィックは Envoy プロキシを経由するため、サービスの変更なしにメッシュ内トラフィックを簡単に制御できます。

本セクションで説明する機能の仕組みに興味がある場合は、[アーキテクチャ概要](/ja/docs/ops/deployment/architecture/)で Istio のトラフィック管理実装の詳細を参照してください。本セクションでは Istio のトラフィック管理機能のみを紹介します。

## Istio トラフィック管理の概要 {#introducing-Istio-traffic-management}

メッシュ内でトラフィックを制御するには、Istio がすべてのエンドポイントの場所と、それらが属するサービスを把握する必要があります。Istio は {{< gloss >}}サービスレジストリ{{</ gloss >}}（サービス登録センター）に接続し、サービスディスカバリシステムと連携します。たとえば、Kubernetes クラスタ上に Istio をインストールすると、Istio はそのクラスタ内のサービスやエンドポイントを自動検出します。

このサービスレジストリを利用して、Envoy プロキシはトラフィックを適切なサービスにルーティングできます。多くのマイクロサービスアプリケーションでは、各サービスのワークロードが複数のインスタンス（負荷分散プール）でトラフィックを処理します。デフォルトでは、Envoy プロキシはラウンドロビン方式でサービスのインスタンス間にトラフィックを分配し、順番に各インスタンスへリクエストを送信します。

Istio の基本的なサービスディスカバリと負荷分散機能だけでもサービスメッシュとして利用できますが、Istio にはさらに多くの機能があります。たとえば、A/B テストの一環として特定の割合のトラフィックを新バージョンのサービスに送ったり、特定のサービスインスタンスサブセットに異なる負荷分散戦略を適用したり、メッシュ外部の依存先をサービスレジストリに追加したりできます。Istio のトラフィック管理 API を使ってトラフィック設定を追加することで、これらすべて、さらにはそれ以上のことが可能です。

他の Istio 設定と同様、これらの API も Kubernetes のカスタムリソース定義（{{< gloss >}}CRD{{</ gloss >}}）で宣言され、YAML で設定できます。

この章の残りでは、各トラフィック管理 API とその使い方を個別に説明します。これらのリソースには以下が含まれます：

- [VirtualService（仮想サービス）](#virtual-services)
- [DestinationRule（目標ルール）](#destination-rules)
- [Gateway（ゲートウェイ）](#gateways)
- [ServiceEntry（サービスエントリ）](#service-entries)
- [Sidecar（サイドカー）](#sidecars)

また、API リソースで構成できる[ネットワークレジリエンスとテスト](#network-resilience-and-testing)についても概説します。

## 仮想サービス {#virtual-services}

[仮想サービス（Virtual Service）](/ja/docs/reference/config/networking/virtual-service/#VirtualService)と[目標ルール（Destination Rule）](#destination-rules)は、Istio のトラフィックルーティング機能の中核となる構成要素です。Istio とプラットフォームが提供する基本的な接続性とサービスディスカバリ機能を基に、仮想サービスを使うことでリクエストがどのサービスにルーティングされるかを柔軟に設定できます。各仮想サービスは順番に評価されるルーティングルールのセットを持ち、Istio は各リクエストを特定の実際のターゲットアドレスにマッチさせます。ユースケースによっては、サービスメッシュ内に複数の仮想サービスを持つことも、仮想サービスが不要な場合もあります。

### なぜ仮想サービスを使うのか？ {#why-use-virtual-services}

仮想サービスは、クライアントリクエストのターゲットアドレスと実際にリクエストを処理するワークロードを分離することで、Istio のトラフィック管理の柔軟性と有効性を大きく高めます。仮想サービスは、これらのワークロードに送信されるトラフィックに対して多様なルーティングルールを指定できます。

なぜこれが重要なのでしょうか？前述の通り、仮想サービスがなければ、Envoy はすべてのサービスインスタンスにラウンドロビンでリクエストを分配します。ワークロードのバージョンなどの知識を活用して、この挙動を改善できます。たとえば、A/B テストで異なるサービスバージョンごとにトラフィックの割合を変えたり、特定のユーザーからのリクエストを特定のインスタンスグループに誘導したりできます。

仮想サービスを使うと、1 つまたは複数のホスト名に対してトラフィックの挙動を指定できます。仮想サービスのルーティングルールで、Envoy に対して仮想サービスのトラフィックをどのターゲットに送るかを指示します。ターゲットは同じサービスの異なるバージョンでも、まったく別のサービスでも構いません。

典型的なユースケースは、サービスの異なるバージョン（サブセット）にトラフィックを分配することです。クライアントは仮想サービスを単一のエンティティとして扱い、リクエストを仮想サービスのホストに送信します。その後、Envoy が仮想サービスのルールに従ってトラフィックを異なるバージョンにルーティングします。たとえば「20% のリクエストを新バージョンへ」「このユーザーのリクエストは v2 へ」などです。これにより、カナリアリリースで新バージョンへのトラフィック割合を段階的に増やすことができます。トラフィックルーティングはインスタンスのデプロイとは独立しているため、新バージョンのインスタンス数をトラフィック量に応じてスケールさせても、ルーティングには影響しません。Kubernetes などのオーケストレーションプラットフォームではインスタンス数に基づくトラフィック分配しかできず、複雑になりがちです。仮想サービスによるカナリアリリースの詳細は[Istio でのカナリアリリース](/ja/blog/2017/0.1-canary/)をご覧ください。

仮想サービスの利点：

- 1 つの仮想サービスで複数のアプリケーションサービスを扱える。Kubernetes を使っている場合、特定のネームスペース内のすべてのサービスを 1 つの仮想サービスで管理できます。単一の仮想サービスを複数の「実サービス」にマッピングすることで、モノリシックアプリをマイクロサービスに段階的に移行する際にも便利です。ルーティングルールで「`monolith.com` への URI は `microservice A` へ」などと指定できます。[下記の例](#more-about-routing-rules)も参照してください。
- [ゲートウェイ](/ja/docs/concepts/traffic-management/#gateways)と連携し、入出力トラフィックの制御ルールを設定できる。

これらの機能を使うには、サービスサブセットの指定などのために目標ルールの設定も必要な場合があります。サブセットやターゲット固有の戦略を別オブジェクトで定義することで、複数の仮想サービス間でルールを再利用しやすくなります。詳細は次章の目標ルールを参照してください。

### 仮想サービスの例 {#virtual-service-example}

以下の仮想サービスは、特定ユーザーからのリクエストをサービスの異なるバージョンにルーティングします。

{{< text yaml >}}
apiVersion: networking.istio.io/v1
kind: VirtualService
metadata:
name: reviews
spec:
hosts:

- reviews
  http:
- match:
  - headers:
    end-user:
    exact: jason
    route:
  - destination:
    host: reviews
    subset: v2
- route: - destination:
  host: reviews
  subset: v3
  {{< /text >}}

#### hosts フィールド {#the-hosts-field}

`hosts` フィールドで仮想サービスのホスト（ユーザーが指定するターゲットやルーティングルールで指定するターゲット）を列挙します。これはクライアントがサービスにリクエストを送る際に使う 1 つ以上のアドレスです。

{{< text yaml >}}
hosts:

- reviews
  {{< /text >}}

仮想サービスのホスト名は IP アドレス、DNS 名、またはプラットフォーム依存の短縮名（Kubernetes サービスの短縮名など）で、FQDN（完全修飾ドメイン名）を暗黙的または明示的に指すことができます。ワイルドカード（「\*」）プレフィックスも使え、複数サービスにマッチするルールも作成可能です。`hosts` フィールドは Istio サービスレジストリに登録されていなくてもよく、仮想的なターゲットアドレスとして使えます。これにより、メッシュ外の仮想ホストもモデル化できます。

#### ルーティングルール {#routing-rules}

`http` フィールドには仮想サービスのルーティングルールが含まれ、マッチ条件やルーティング動作を記述します。これにより HTTP/1.1、HTTP2、gRPC などのトラフィックを hosts で指定したターゲットに送信できます（`tcp` や `tls` セクションで [TCP](/ja/docs/reference/config/networking/virtual-service/#TCPRoute) や未終端 [TLS](/ja/docs/reference/config/networking/virtual-service/#TLSRoute) トラフィックのルールも設定可能）。ルーティングルールは、リクエストをどのターゲットアドレスに送るかを指定し、0 個以上のマッチ条件を持てます。

##### マッチ条件 {#match-condition}

例の最初のルーティングルールには 1 つの条件があり、`match` フィールドで始まります。この例では「jason」ユーザーからのリクエストにこのルールを適用したいので、`headers`、`end-user`、`exact` で該当リクエストを選択します。

{{< text yaml >}}

- match:
  - headers:
    end-user:
    exact: jason
    {{< /text >}}

##### Destination {#destination}

`route` セクションの `destination` フィールドで、条件に一致したトラフィックの実際のターゲットアドレスを指定します。仮想サービスの `hosts` と異なり、destination の host は Istio サービスレジストリに存在する実際のターゲットでなければなりません。プロキシ付きサービスやサービスエントリで追加した外部サービスなどが該当します。この例では Kubernetes サービス名を使っています：

{{< text yaml >}}
route:

- destination:
  host: reviews
  subset: v2
  {{< /text >}}

この例や他の例では簡単のため Kubernetes の短縮名を使っていますが、Istio は仮想サービスのネームスペースに基づいて FQDN を補完します。短縮名はどのネームスペースでも使えますが、

{{< warning >}}
Kubernetes の短縮名は、ターゲットホストと仮想サービスが同じネームスペースにある場合のみ有効です。短縮名は設定ミスの原因になりやすいため、本番環境では FQDN の指定を推奨します。
{{< /warning >}}

destination セクションでは Kubernetes サービスのサブセットも指定でき、条件に一致したリクエストをそのサブセットにルーティングします。この例ではサブセット名は v2 です。サブセットの定義方法は[目標ルール](#destination-rules)で説明します。

#### ルーティングルールの優先順位 {#routing-rule-precedence}

**ルーティングルール**は上から順に評価され、仮想サービスで最初に定義されたルールが最も優先されます。この例では、最初のルールに一致しないトラフィックは 2 番目のルール（デフォルトターゲット）に送られます。2 番目のルールは match 条件がなく、すべてのトラフィックを v3 サブセットに送ります。

{{< text yaml >}}

- route:
  - destination:
    host: reviews
    subset: v3
    {{< /text >}}

各仮想サービスの最後に「無条件」または重み付きのデフォルトルールを追加することを推奨します。これにより、すべてのトラフィックが必ず何らかのルールに一致します。

### ルーティングルールの詳細 {#more-about-routing-rules}

ルーティングルールは、特定のトラフィックサブセットを指定ターゲットにルーティングする強力なツールです。トラフィックのポート、ヘッダー、URI などでマッチ条件を設定できます。たとえば、次の仮想サービスは `http://bookinfo.com/` の一部として ratings と reviews という 2 つの独立したサービスにリクエストを送ります。ルールは URI に基づいて適切なサービスにトラフィックを振り分けます。

{{< text yaml >}}
apiVersion: networking.istio.io/v1
kind: VirtualService
metadata:
name: bookinfo
spec:
hosts: - bookinfo.com
http:

- match:
  - uri:
    prefix: /reviews
    route:
  - destination:
    host: reviews
- match: - uri:
  prefix: /ratings
  route: - destination:
  host: ratings
  {{< /text >}}

マッチ条件にはプレフィックスや正規表現なども使えます。

同じ `match` ブロックに複数の条件を AND で追加したり、1 つのルールに複数の `match` ブロックを OR で追加したりできます。仮想サービスごとに複数のルーティングルールも設定可能です。これにより、ルーティング条件の複雑さを自由に調整できます。マッチ条件や値の詳細は [`HTTPMatchRequest` リファレンス](/ja/docs/reference/config/networking/virtual-service/#HTTPMatchRequest) を参照してください。

また、マッチ条件を使ってリクエストをパーセンテージ（重み）で分割することもできます。これは A/B テストやカナリアリリースに便利です：

{{< text yaml >}}
spec:
hosts:

- reviews
  http:
- route: - destination:
  host: reviews
  subset: v1
  weight: 75 - destination:
  host: reviews
  subset: v2
  weight: 25
  {{< /text >}}

ルーティングルールでは、トラフィックに対して以下のような操作も可能です：

- ヘッダーの追加・削除
- URL の書き換え
- ターゲットへのリクエストに[リトライポリシー](#retries)を設定

これらの操作の詳細は [`HTTPRoute` リファレンス](/ja/docs/reference/config/networking/virtual-service/#HTTPRoute) を参照してください。

## 目標ルール {#destination-rules}

[仮想サービス](#virtual-services)と同様に、[目標ルール](/ja/docs/reference/config/networking/destination-rule/#DestinationRule)も Istio のトラフィックルーティングの重要な構成要素です。仮想サービスがトラフィックをどのターゲットにルーティングするかを定義するのに対し、目標ルールはそのターゲットのトラフィック制御を設定します。仮想サービスのルーティングルール評価後、目標ルールが「実際の」ターゲットアドレスに適用されます。

特に、目標ルールを使うと、バージョンごとなどでサービスインスタンスをサブセット化し、仮想サービスのルーティングルールでこれらサブセットを使ってトラフィックを制御できます。

また、目標ルールでは、ターゲットサービスやサブセットごとに Envoy のトラフィックポリシー（負荷分散モデル、TLS モード、サーキットブレーカーなど）をカスタマイズできます。詳細は[目標ルールリファレンス](/ja/docs/reference/config/networking/destination-rule/)を参照してください。

### 負荷分散オプション {#load-balancing-options}

デフォルトでは、Istio はラウンドロビン方式の負荷分散を行い、インスタンスプール内の各インスタンスに順番にリクエストを送ります。`DestinationRule` で特定サービスやサブセットへのトラフィックに対して以下の負荷分散モデルを指定できます。

- ランダム：リクエストをランダムにインスタンスへ転送
- 重み付き：指定したパーセンテージでインスタンスへ転送
- 最小リクエスト：最もリクエスト数が少ないインスタンスへ転送
- 一貫性ハッシュ：HTTP ヘッダーや Cookie などでソフトなセッションアフィニティを実現
- リングハッシュ：[Ketama アルゴリズム](https://www.metabrew.com/article/libketama-consistent-hashing-algo-memcached-clients)による一貫性ハッシュ
- Maglev：[Maglev 論文](https://research.google/pubs/maglev-a-fast-and-reliable-software-network-load-balancer/)に基づく一貫性ハッシュ

詳細は [Envoy の負荷分散ドキュメント](https://www.envoyproxy.io/docs/envoy/latest/intro/arch_overview/upstream/load_balancing/load_balancers) を参照してください。

### 目標ルールの例 {#destination-rule-example}

以下の例では、`my-svc` サービスに対して異なる負荷分散戦略を持つ 3 つのサブセットを設定しています：

{{< text yaml >}}
apiVersion: networking.istio.io/v1
kind: DestinationRule
metadata:
name: my-destination-rule
spec:
host: my-svc
trafficPolicy:
loadBalancer:
simple: RANDOM
subsets:

- name: v1
  labels:
  version: v1
- name: v2
  labels:
  version: v2
  trafficPolicy:
  loadBalancer:
  simple: ROUND_ROBIN
- name: v3
  labels:
  version: v3
  {{< /text >}}

各サブセットは 1 つ以上の `labels` で定義され、Kubernetes では Pod などのオブジェクトに付与されるキー/バリューのペアです。これらのラベルは Kubernetes サービスの Deployment に適用され、`metadata` で異なるバージョンを識別します。

サブセットの定義に加え、この目標ルールはすべてのサブセットにデフォルトのトラフィックポリシーを適用し、v2 サブセットには個別のポリシーで上書きしています。

## ゲートウェイ {#gateways}

[ゲートウェイ](/ja/docs/reference/config/networking/gateway/#Gateway)を使うと、メッシュの入出力トラフィックを制御できます。ゲートウェイ設定はメッシュのエッジで動作する独立した Envoy プロキシに適用され、サービスワークロードとともに動作する Sidecar Envoy とは異なります。

Kubernetes Ingress API などの他のトラフィック制御メカニズムと異なり、Istio のゲートウェイはトラフィックルーティングの強力な機能と柔軟性を最大限に活用できます。Istio のゲートウェイリソースでは L4 ～ L6 の負荷分散属性（公開ポートや TLS 設定など）を設定でき、L7 のルーティングは通常の [仮想サービス](#virtual-services) で管理します。これにより、他のデータプレーンと同様にゲートウェイトラフィックも管理できます。

ゲートウェイは主にインバウンドトラフィックの管理に使いますが、アウトバウンドゲートウェイも設定できます。アウトバウンドゲートウェイを使うと、メッシュ外へのトラフィックを専用ノード経由に限定したり、外部ネットワークへのアクセス制御や[出口トラフィックのセキュリティ強化](/ja/blog/2019/egress-traffic-control-in-istio-part-1/)が可能です。内部専用プロキシとしても利用できます。

Istio には事前構成済みのゲートウェイデプロイメント（`istio-ingressgateway` と `istio-egressgateway`）が用意されており、[デモ構成ファイル](/ja/docs/setup/getting-started/)で両方、[default 構成ファイル](/ja/docs/setup/additional-setup/config-profiles/)では入口ゲートウェイのみがデプロイされます。これらのデプロイメントに独自のゲートウェイ設定を適用したり、独自のゲートウェイプロキシを構成したりできます。

### Gateway の例 {#gateway-example}

以下は外部 HTTPS インバウンドトラフィック用のゲートウェイ設定例です：

{{< text yaml >}}
apiVersion: networking.istio.io/v1
kind: Gateway
metadata:
name: ext-host-gwy
spec:
selector:
app: my-gateway-controller
servers:

- port:
  number: 443
  name: https
  protocol: HTTPS
  hosts: - ext-host.example.com
  tls:
  mode: SIMPLE
  credentialName: ext-host-cert
  {{< /text >}}

このゲートウェイ設定により、`ext-host.example.com` への HTTPS トラフィックが 443 番ポートでメッシュに流入しますが、リクエストのルーティングルールはまだ指定されていません。ルーティングを指定しゲートウェイを機能させるには、ゲートウェイを仮想サービスにバインドする必要があります。以下の例のように、仮想サービスの `gateways` フィールドで設定します：

{{< text yaml >}}
apiVersion: networking.istio.io/v1
kind: VirtualService
metadata:
name: virtual-svc
spec:
hosts:

- ext-host.example.com
  gateways: - ext-host-gwy
  {{< /text >}}

この後、アウトバウンドトラフィックにもルーティングルール付きの仮想サービスを設定できます。

## サービスエントリ {#service-entries}

[サービスエントリ（Service Entry）](/ja/docs/reference/config/networking/service-entry/#ServiceEntry)を使うと、Istio が内部で管理するサービスレジストリに新たなエントリを追加できます。サービスエントリを追加すると、Envoy プロキシはそのサービスをメッシュ内サービスのように扱い、トラフィックを送信できます。サービスエントリを設定することで、メッシュ外サービスへのトラフィックも制御でき、以下のようなことが可能です：

- 外部ターゲットへのリダイレクトやフォワード（Web API コールやレガシーシステムへのトラフィックなど）
- 外部ターゲットに対する[リトライ](#retries)、[タイムアウト](#timeouts)、[フォールトインジェクション](#fault-injection)の設定
- 仮想マシン上のサービスを[メッシュに追加](/ja/docs/examples/virtual-machines/single-network/#running-services-on-the-added-VM)

すべての外部サービスにサービスエントリを追加する必要はありません。デフォルトでは、Istio は未知のサービスへのリクエストをそのまま転送します。ただし、サービスレジストリに登録されていないターゲットへのトラフィックは Istio の機能で制御できません。

### サービスエントリの例 {#service-entry-example}

以下の mesh-external サービスエントリは、`ext-svc.example.com` という外部依存先を Istio のサービスレジストリに追加します：

{{< text yaml >}}
apiVersion: networking.istio.io/v1
kind: ServiceEntry
metadata:
name: svc-entry
spec:
hosts:

- ext-svc.example.com
  ports:
- number: 443
  name: https
  protocol: HTTPS
  location: MESH_EXTERNAL
  resolution: DNS
  {{< /text >}}

外部リソースは `hosts` フィールドで指定します。FQDN やワイルドカードプレフィックスも利用可能です。

サービスエントリへのトラフィックも仮想サービスや目標ルールで細かく制御できます。たとえば、以下の目標ルールはサービスエントリで設定した `ext-svc.example.com` 外部サービスの接続タイムアウトを調整しています：

{{< text yaml >}}
apiVersion: networking.istio.io/v1
kind: DestinationRule
metadata:
name: ext-res-dr
spec:
host: ext-svc.example.com
trafficPolicy:
connectionPool:
tcp:
connectTimeout: 1s
{{< /text >}}

詳細は[サービスエントリリファレンス](/ja/docs/reference/config/networking/service-entry)を参照してください。

## Sidecar {#sidecars}

デフォルトでは、Istio は各 Envoy プロキシが関連ワークロードのすべてのポートへのリクエストを受け付け、適切なワークロードに転送できるようにします。[Sidecar](/ja/docs/reference/config/networking/sidecar/#Sidecar) 設定を使うと、以下のことが可能です：

- Envoy プロキシが受け付けるポートやプロトコルの微調整
- Envoy プロキシがアクセスできるサービスの範囲を制限

大規模なアプリケーションでは、Sidecar の到達範囲を制限することで、各プロキシがメッシュ内のすべてのサービスにアクセスできる場合に比べてメモリ使用量を抑え、パフォーマンスを向上できます。

Sidecar 設定は特定のネームスペース内のすべてのワークロードに適用したり、`workloadSelector` で特定ワークロードだけに適用したりできます。以下の例では、`bookinfo` ネームスペース内のすべてのサービスが同じネームスペースと Istio コントロールプレーン（Egress やテレメトリ機能用）内のサービスだけにアクセスできるようにしています：

{{< text yaml >}}
apiVersion: networking.istio.io/v1
kind: Sidecar
metadata:
name: default
namespace: bookinfo
spec:
egress:

- hosts: - "./_" - "istio-system/_"
  {{< /text >}}

詳細は [Sidecar リファレンス](/ja/docs/reference/config/networking/sidecar/) を参照してください。

## ネットワークレジリエンスとテスト {#network-resilience-and-testing}

トラフィック制御に加え、Istio には障害復旧やフォールトインジェクションの機能もあり、これらはランタイムで動的に設定できます。これらの機能を使うことで、アプリケーションの安定稼働や、サービスメッシュが障害ノードに耐え、障害の連鎖を防ぐことができます。

### タイムアウト {#timeouts}

タイムアウトは、Envoy プロキシがサービスからの応答を待つ最大時間を指定し、サービスが無限に応答を待たされることを防ぎます。HTTP リクエストのデフォルトタイムアウトは 15 秒で、15 秒以内に応答がなければ呼び出しは失敗します。

アプリやサービスによっては、Istio のデフォルトタイムアウトが適切でない場合もあります。タイムアウトが長すぎると失敗サービスの応答待ちで遅延が増え、短すぎると複数サービスをまたぐ操作で不要な失敗が発生します。最適なタイムアウトを見つけて使うために、Istio では[仮想サービス](#virtual-services)でサービスごとにタイムアウトを動的に調整できます。以下は ratings サービスの v1 サブセットへの呼び出しに 10 秒のタイムアウトを指定する例です：

{{< text yaml >}}
apiVersion: networking.istio.io/v1
kind: VirtualService
metadata:
name: ratings
spec:
hosts:

- ratings
  http:
- route: - destination:
  host: ratings
  subset: v1
  timeout: 10s
  {{< /text >}}

### リトライ {#retries}

リトライ設定では、初回呼び出しが失敗した場合に Envoy プロキシがサービスへの接続を再試行する最大回数を指定します。リトライは、一時的な障害（サービスやネットワークの一時的な過負荷など）による恒久的な失敗を防ぎ、サービスの可用性やアプリのパフォーマンスを向上させます。リトライ間隔（25ms 以上）は可変で、Istio が自動的に決定し、呼び出し先サービスの過負荷を防ぎます。HTTP リクエストのデフォルトリトライ回数は 2 回です。

タイムアウトと同様、Istio のデフォルトリトライ動作がアプリの要件に合わない場合もあります（失敗サービスへの過剰なリトライで遅延が増えるなど）。[仮想サービス](#virtual-services)でサービスごとにリトライ設定を調整でき、アプリコードの変更は不要です。各リトライのタイムアウトも指定でき、リトライごとにサービス接続を待つ時間を細かく制御できます。以下は初回失敗後に最大 3 回リトライし、各リトライのタイムアウトを 2 秒に設定する例です：

{{< text yaml >}}
apiVersion: networking.istio.io/v1
kind: VirtualService
metadata:
name: ratings
spec:
hosts:

- ratings
  http:
- route: - destination:
  host: ratings
  subset: v1
  retries:
  attempts: 3
  perTryTimeout: 2s
  {{< /text >}}

### サーキットブレーカー {#circuit-breakers}

サーキットブレーカーは、Istio がレジリエントなマイクロサービスアプリを構築するための有用な仕組みです。サーキットブレーカーでは、サービスの各ホストへの呼び出しに制限（同時接続数や失敗回数など）を設け、制限を超えると「遮断」してそのホストへの接続を停止します。これにより、過負荷や障害ホストへの接続をクライアントが繰り返すことなく、すぐに失敗させることができます。

サーキットブレーカーは負荷分散プール内の「実際の」ターゲットアドレスに適用され、[目標ルール](#destination-rules)で各ホストごとに閾値を設定できます。以下は reviews サービスの v1 サブセットのワークロードで同時接続数を 100 に制限する例です：

{{< text yaml >}}
apiVersion: networking.istio.io/v1
kind: DestinationRule
metadata:
name: reviews
spec:
host: reviews
subsets:

- name: v1
  labels:
  version: v1
  trafficPolicy:
  connectionPool:
  tcp:
  maxConnections: 100
  {{< /text >}}

詳細は[サーキットブレーカー](/ja/docs/tasks/traffic-management/circuit-breaking/)を参照してください。

### フォールトインジェクション {#fault-injection}

ネットワークや障害復旧戦略を設定した後、Istio のフォールトインジェクション機能でアプリ全体の障害復旧力をテストできます。フォールトインジェクションは、システムに意図的に障害を注入し、システムが障害から回復できるかを検証するテスト手法です。これにより、障害復旧戦略が互換性を損なったり厳しすぎて重要サービスが利用不能になることを防げます。

{{< warning >}}
現時点では、同じ仮想サービスでフォールトインジェクションとリトライやタイムアウト設定を併用できません。詳細は[トラフィック管理の問題](/ja/docs/ops/common-problems/network-issues/#virtual-service-with-fault-injection-and-retrytimeout-policies-not-working-as-expected)を参照してください。
{{< /warning >}}

他の障害注入手法（パケット遅延やネットワーク層での Pod 強制終了など）と異なり、Istio ではアプリケーション層で障害を注入できます。これにより、HTTP エラーコードなど、より意味のある障害を注入し、現実的なテストが可能です。

フォールトインジェクションには 2 種類あり、どちらも[仮想サービス](#virtual-services)で設定します：

- 遅延：ネットワーク遅延や過負荷サービスを模擬する時間的障害
- 終了：上流サービスの障害を模擬するクラッシュ障害。通常は HTTP エラーコードや TCP 接続失敗として現れます。

以下の仮想サービスは、ratings サービスへのリクエストの 0.1% に 5 秒の遅延を注入します：

{{< text yaml >}}
apiVersion: networking.istio.io/v1
kind: VirtualService
metadata:
name: ratings
spec:
hosts:

- ratings
  http:
- fault:
  delay:
  percentage:
  value: 0.1
  fixedDelay: 5s
  route: - destination:
  host: ratings
  subset: v1
  {{< /text >}}

遅延や終了の詳細な設定方法は[フォールトインジェクション](/ja/docs/tasks/traffic-management/fault-injection/)を参照してください。

### アプリケーションとの連携 {#working-with-your-applications}

Istio の障害復旧機能はアプリケーションからは完全に透過的です。レスポンスが返るまで、アプリケーションは Envoy Sidecar プロキシが障害を処理しているかどうかを知りません。したがって、アプリケーションコードで障害復旧戦略を設定している場合は、両者が独立して動作することを考慮しないと競合が発生します。たとえば、アプリで 2 秒のタイムアウトを設定し、仮想サービスで 3 秒のタイムアウトとリトライを設定した場合、アプリのタイムアウトが先に発動し、Envoy のタイムアウトやリトライは無効になります。

Istio の障害復旧機能はサービスの信頼性と可用性を高めますが、アプリケーション側でも障害やエラーを適切に処理し、フォールバック動作を実装する必要があります。たとえば、負荷分散先のすべてのインスタンスが失敗した場合、Envoy は `HTTP 503` を返します。アプリケーションは `HTTP 503` エラーコードを受けてフォールバック処理を実装する必要があります。
