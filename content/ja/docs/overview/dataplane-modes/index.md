---
title: Sidecar か Ambient か？
description: Istio の 2 つのデータプレーンモードと、どちらを使うべきかを解説します。
weight: 30
keywords: [sidecar, ambient]
owner: istio/wg-docs-maintainers-english
test: n/a
---

Istio サービスメッシュは論理的にデータプレーンとコントロールプレーンに分かれます。

{{< gloss "data plane" >}}データプレーン{{< /gloss >}}は、マイクロサービス間のすべてのネットワーク通信を仲介・制御するプロキシ群です。
これらのプロキシは、すべてのメッシュトラフィックの可観測データも収集・報告します。

{{< gloss "control plane" >}}コントロールプレーン{{< /gloss >}}は、これらのプロキシの管理・構成を担います。

Istio では主に 2 種類の{{< gloss "data plane mode">}}データプレーンモード{{< /gloss >}}をサポートしています：

- **Sidecar モード**：クラスタ内の各 Pod に Envoy プロキシをデプロイ、または VM 上のサービスと並行して実行
- **Ambient モード**：各ノードに L4 プロキシを配置し、必要に応じて各ネームスペースに Envoy プロキシを追加して L7 機能を実現

どちらのモードも、ネームスペースやワークロード単位で選択できます。

## Sidecar モード {#sidecar=mode}

Istio は 2017 年の初リリース以来、Sidecar モードをベースに構築されてきました。
Sidecar モードは分かりやすく実績も豊富ですが、リソースコストや運用負荷がかかります。

- デプロイした各アプリケーションに Envoy プロキシが Sidecar として{{< gloss "injection" >}}注入{{< /gloss >}}されます
- すべてのプロキシが L4/L7 トラフィックを処理できます

## Ambient モード {#ambient-mode}

Ambient モードは 2022 年に登場し、Sidecar モードの課題を解決するために設計されました。
Istio 1.22 以降、単一クラスタでの利用が本番対応となっています。

- すべてのトラフィックはノード上の L4 プロキシで処理されます
- 必要に応じて Envoy プロキシを追加し、L7 機能を有効化できます

## Sidecar と Ambient の選択 {#choosing-between-sidecar-and-ambient}

多くのユーザーはまずゼロトラストセキュリティのためにメッシュを導入し、必要に応じて L7 機能を選択的に有効化します。
Ambient メッシュでは、L7 機能が不要な場合は L7 処理コストを完全に回避できます。

<table>
  <thead>
    <tr>
      <td style="border-width: 0px"></td>
      <th><strong>Sidecar</strong></th>
      <th><strong>Ambient</strong></th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>トラフィック管理</th>
      <td>Istio の全機能</td>
      <td>Istio の全機能（waypoint 利用時）</td>
    </tr>
    <tr>
      <th>セキュリティ</th>
      <td>Istio の全機能</td>
      <td>Istio の全機能：Ambient では暗号化と L4 認可。L7 認可は waypoint が必要。</td>
    </tr>
    <tr>
      <th>可観測性</th>
      <td>Istio の全機能</td>
      <td>Istio の全機能：Ambient で L4 テレメトリー、waypoint で L7 可観測性</td>
    </tr>
    <tr>
      <th>拡張性</th>
      <td>Istio の全機能</td>
      <td><a href="/ja/docs/ambient/usage/extend-waypoint-wasm">WebAssembly プラグイン</a>（waypoint 利用時）<br>EnvoyFilter API は非対応</td>
    </tr>
    <tr>
      <th>ワークロード追加</th>
      <td>ネームスペースにラベル付与し、全 Pod を再起動して Sidecar を追加</td>
      <td>ネームスペースにラベル付与のみ - Pod 再起動不要</td>
    </tr>
    <tr>
      <th>段階的導入</th>
      <td>バイナリ：Sidecar の有無</td>
      <td>段階的：L4 は常時有効、L7 は設定で追加</td>
    </tr>
    <tr>
      <th>ライフサイクル管理</th>
      <td>アプリ開発者が管理</td>
      <td>プラットフォーム管理者</td>
    </tr>
    <tr>
      <th>リソース利用</th>
      <td>非効率：各 Pod ごとに最大リソースを見積もる必要あり</td>
      <td>waypoint プロキシは通常の Deployment と同様に自動スケール可能。複数レプリカのワークロードは 1 つの waypoint を共有できる</td>
    </tr>
    <tr>
      <th>平均リソースコスト</th>
      <td>大</td>
      <td>小</td>
    </tr>
    <tr>
      <th>平均遅延（p90/p99）</th>
      <td>0.63ms-0.88ms</td>
      <td>Ambient：0.16ms-0.20ms<br />waypoint：0.40ms-0.50ms</td>
    </tr>
    <tr>
      <th>L7 処理ステップ</th>
      <td>2 ステップ（送信元・宛先 Sidecar）</td>
      <td>1 ステップ（宛先 waypoint）</td>
    </tr>
    <tr>
      <th>大規模構成</th>
      <td><a href="/ja/docs/ops/configuration/mesh/configuration-scoping/">各 Sidecar のスコープ設定</a>が必要</td>
      <td>カスタム設定不要</td>
    </tr>
    <tr>
      <th>サーバーファーストプロトコル対応</th>
      <td><a href="/ja/docs/ops/deployment/application-requirements/#server-first-protocols">設定が必要</a></td>
      <td>対応</td>
    </tr>
    <tr>
      <th>Kubernetes Job 対応</th>
      <td>Sidecar の寿命管理が複雑</td>
      <td>透過的に対応</td>
    </tr>
    <tr>
      <th>セキュリティモデル</th>
      <td>最強：各ワークロードごとに鍵を発行</td>
      <td>強：ノードごとに鍵を発行</td>
    </tr>
    <tr>
      <th>侵害された Pod の鍵アクセス</th>
      <td>可能</td>
      <td>不可</td>
    </tr>
    <tr>
      <th>サポート</th>
      <td>安定版、マルチクラスタ対応</td>
      <td>安定版、単一クラスタのみ</td>
    </tr>
    <tr>
      <th>対応プラットフォーム</th>
      <td>Kubernetes（任意 CNI）、VM</td>
      <td>Kubernetes（任意 CNI）</td>
    </tr>
  </tbody>
</table>

## L4 と L7 の機能 {#layer-4-vs-layer-7-features}

L7 プロトコル処理は L4 のネットワークパケット処理よりも大きなオーバーヘッドがあります。
要件が L4 で満たせる場合、サービスメッシュのコストを大幅に削減できます。

### セキュリティ {#security}

<table>
  <thead>
    <tr>
      <td style="border-width: 0px" width="20%"></td>
      <th width="40%">L4</th>
      <th width="40%">L7</th>
    </tr>
   </thead>
   <tbody>
    <tr>
      <th>暗号化</th>
      <td>すべての Pod 間トラフィックは {{< gloss "mutual tls authentication" >}}mTLS{{< /gloss >}} で暗号化</td>
      <td>該当なし。Istio のサービス ID は TLS ベース</td>
    </tr>
    <tr>
      <th>サービス間認証</th>
      <td>mTLS 証明書による {{< gloss >}}SPIFFE{{< /gloss >}}。Istio は短期 X.509 証明書で Pod のサービスアカウント ID をエンコード</td>
      <td>該当なし。Istio のサービス ID は TLS ベース</td>
    </tr>
    <tr>
      <th>サービス間認可</th>
      <td>ネットワークベースの認可＋ID ベースのポリシー（例：A は 10.2.0.0/16 からのみ受信可、A は B へ通信可）</td>
      <td>完全なポリシー（例：A は READ スコープのユーザーのみ B の GET /foo を許可）</td>
    </tr>
    <tr>
      <th>エンドユーザー認証</th>
      <td>該当なし。ユーザー単位の設定は不可</td>
      <td>JWT のローカル認証、OAuth/OIDC 連携によるリモート認証</td>
    </tr>
    <tr>
      <th>エンドユーザー認可</th>
      <td>該当なし。同上</td>
      <td><a href="/ja/docs/reference/config/security/conditions/">特定スコープ・発行者・サブジェクト・オーディエンス等のユーザー認可</a>が可能。外部認可（OPA など）も利用可</td>
    </tr>
  </tbody>
</table>

### 可観測性 {#observability}

<table>
  <thead>
    <tr>
      <td style="border-width: 0px" width="20%"></td>
      <th width="40%">L4</th>
      <th width="40%">L7</th>
    </tr>
   </thead>
   <tbody>
    <tr>
      <th>ログ</th>
      <td>基本的なネットワーク情報（5 タプル、送受信バイト数など）。<a href="https://www.envoyproxy.io/docs/envoy/latest/configuration/observability/access_log/usage#command-operators">Envoy ドキュメント</a>参照</td>
      <td><a href="https://www.envoyproxy.io/docs/envoy/latest/configuration/observability/access_log/usage#command-operators">リクエストメタデータを含む完全なログ</a>＋基本ネットワーク情報</td>
    </tr>
    <tr>
      <th>トレース</th>
      <td>現時点では不可。将来的に HBONE で実現予定</td>
      <td>Envoy による分散トレース。<a href="/ja/docs/tasks/observability/distributed-tracing/overview/">Istio トレース概要</a>参照</td>
    </tr>
    <tr>
      <th>メトリクス</th>
      <td>TCP のみ（送受信バイト数、パケット数など）</td>
      <td>L7 RED メトリクス（リクエスト数、エラー率、レイテンシ）</td>
    </tr>
  </tbody>
</table>

### トラフィック管理 {#traffic-management}

<table>
  <thead>
    <tr>
      <td style="border-width: 0px" width="20%"></td>
      <th width="40%">L4</th>
      <th width="40%">L7</th>
    </tr>
   </thead>
   <tbody>
    <tr>
      <th>ロードバランシング</th>
      <td>コネクション単位のみ。<a href="/ja/docs/tasks/traffic-management/tcp-traffic-shifting/">TCP トラフィックシフト</a>参照</td>
      <td>リクエスト単位。カナリアリリースや gRPC など。<a href="/ja/docs/tasks/traffic-management/traffic-shifting/">HTTP トラフィックシフト</a>参照</td>
    </tr>
    <tr>
      <th>サーキットブレーカー</th>
      <td><a href="/ja/docs/reference/config/networking/destination-rule/#ConnectionPoolSettings-TCPSettings">TCP のみ</a></td>
      <td>TCP に加え、<a href="/ja/docs/reference/config/networking/destination-rule/#ConnectionPoolSettings-HTTPSettings">HTTP 設定</a>も利用可</td>
    </tr>
    <tr>
      <th>異常検知</th>
      <td>コネクション確立・失敗時</td>
      <td>リクエスト成功・失敗時</td>
    </tr>
    <tr>
      <th>レート制限</th>
      <td>グローバル・ローカルレート制限。<a href="https://www.envoyproxy.io/docs/envoy/latest/configuration/listeners/network_filters/rate_limit_filter#config-network-filters-rate-limit">L4 コネクション単位</a></td>
      <td>リクエスト単位。<a href="https://www.envoyproxy.io/docs/envoy/latest/configuration/http/http_filters/rate_limit_filter#config-http-filters-rate-limit">L7 メタデータ単位</a></td>
    </tr>
    <tr>
      <th>タイムアウト</th>
      <td>コネクション確立時のみ（サーキットブレーカーで設定）</td>
      <td>リクエスト単位</td>
    </tr>
    <tr>
      <th>リトライ</th>
      <td>コネクション確立時にリトライ</td>
      <td>リクエスト失敗時にリトライ</td>
    </tr>
    <tr>
      <th>フォールトインジェクション</th>
      <td>不可。TCP では設定不可</td>
      <td>アプリ・コネクションレベルの障害注入（<a href="/ja/docs/tasks/traffic-management/fault-injection/">タイムアウト、遅延、特定レスポンスコード</a>）</td>
    </tr>
    <tr>
      <th>トラフィックミラーリング</th>
      <td>不可。HTTP のみ対応</td>
      <td><a href="/ja/docs/tasks/traffic-management/mirroring/">リクエストを複数バックエンドにミラー</a></td>
    </tr>
  </tbody>
</table>

## 未対応機能 {#unsupported-features}

以下の機能は Sidecar モードでは利用できますが、Ambient モードでは未対応です：

- Sidecar と waypoint の相互運用
- マルチクラスタ
- マルチネットワーク
- VM サポート
