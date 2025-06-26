---
title: Ambient データプレーン
description: Ambient データプレーンが Ambient メッシュ内のワークロード間でどのようにトラフィックをルーティングするかを理解します。
weight: 2
owner: istio/wg-networking-maintainers
test: no
---

{{< gloss "ambient" >}}Ambient モード{{< /gloss >}}では、ワークロードは 3 つのカテゴリに分類されます：

1. **メッシュ外**：メッシュ機能が有効になっていない標準の Pod。
   Istio と Ambient の {{< gloss "data plane" >}}データプレーン{{< /gloss >}} は両方とも有効になっていません。
1. **メッシュ内**：Ambient {{< gloss "data plane" >}}データプレーン{{< /gloss >}} に含まれる Pod。
   {{< gloss >}}ztunnel{{< /gloss >}} によって 4 層レベルでトラフィックをインターセプトします。
   このモードでは、Pod トラフィックに対して L4 ポリシーを適用できます。
   `istio.io/dataplane-mode=ambient` ラベルを設定することでこのモードを有効にできます。
   詳細については、[ラベル](/ja/docs/ambient/architecture#ambient-labels)を参照してください。
1. **メッシュ内で waypoint が有効になっている**：**メッシュ内で** {{< gloss "waypoint" >}}waypoint プロキシ{{< /gloss >}} がデプロイされています。
   このモードでは、Pod トラフィックに対して L7 ポリシーを適用できます。
   `istio.io/use-waypoint` ラベルを設定することでこのモードを有効にできます。
   詳細については、[ラベル](/ja/docs/ambient/architecture#ambient-labels)を参照してください。

ワークロードの所属するカテゴリによって、トラフィックのパスは異なります。

## メッシュ内ルーティング {#in-mesh-routing}

### 出站 {#outbound}

Ambient メッシュ内の Pod が出站リクエストを送信すると、
それは[透過的に](/ja/docs/ambient/architecture/traffic-redirection)ノードローカルの ztunnel にリダイレクトされます。
この ztunnel は、リクエストをどこに転送するか、およびどのように転送するかを決定します。
一般的に、トラフィックルーティングの動作は Kubernetes のデフォルトのトラフィックルーティングと同じです。
`Service` 
に対するリクエストは `Service` 内のエンドポイントに送信され、
直接 `Pod` IP に対するリクエストはその IP に直接送信されます。

しかし、目的地の権限によって、異なる動作が発生する可能性があります。
もし目的地がメッシュに追加されている、または他の方法で Istio プロキシに権限を付与されている（例えば Sidecar）場合、
リクエストは暗号化された {{< gloss "HBONE" >}}HBONE トンネル{{< /gloss >}} にアップグレードされます。
もし目的地に waypoint プロキシがある場合、HBONE にアップグレードするだけでなく、
該当の waypoint にリクエストが転送されて L7 ポリシーが実行されます。

{{< tip >}}
注意：`Service` に対するリクエストの場合、もし該サービスが **waypoint を持っている** 場合、
そのリクエストはその waypoint に転送されて L7 ポリシーが実行されます。
同様に、`Pod` IP に対するリクエストの場合、もし Pod が **waypoint を持っている** 場合、
そのリクエストはその waypoint に転送されて L7 ポリシーが実行されます。
`Deployment` 中の Pod に関連付けられたラベルを変更することができるため、
技術的には、一部の Pod は waypoint を使用できますが、他の Pod は使用できません。
通常、ユーザーはこの高度なユースケースを避けることをお勧めします。
{{< /tip >}}

### 入站 {#inbound}

Ambient メッシュ内の Pod が入站リクエストを受信すると、
それは[透過的に](/ja/docs/ambient/architecture/traffic-redirection)ノードローカルの ztunnel にリダイレクトされます。
この ztunnel は、リクエストをどこに転送するか、およびどのように転送するかを決定します。

Pod は HBONE トラフィックまたは明文トラフィックを受信できます。
これらのトラフィックはデフォルトで ztunnel によって受け入れられます。
グリッド外からのリクエストは認証ポリシーの評価時に対等な ID を持たないため、
ユーザーは明文トラフィックをすべてブロックするように要求するポリシーを設定できます（**任意**の認証または特定の認証）。

もし目的地が waypoint を有効にしている場合、もしソースが Ambient メッシュ内にある場合、
ソースの ztunnel はリクエストが**waypoint によって実行される**ことを確認します。
しかし、グリッド外のワークロードは waypoint プロキシについて何も知らないため、
目的地が waypoint を有効にしていても、リクエストは直接目的地に送信され、waypoint プロキシを通過しません。
現在、Sidecar とゲートウェイからのトラフィックは waypoint プロキシを通過しません。
また、将来のバージョンでは waypoint プロキシを感知するようになります。

#### データプレーンの詳細 {#dataplane-details}

##### アイデンティティ {#identity}

Ambient メッシュ内のワークロード間のすべての入站および出站 L4 TCP トラフィックは、
{{< gloss "HBONE" >}}HBONE{{< /gloss >}}、ztunnel、および x509 証明書を使用して mTLS で保護されます。

{{< gloss "mutual tls authentication" >}}mTLS{{< /gloss >}} の強制要求により、
ソースと目的地は一意の x509 アイデンティティを持ち、それらを使用して接続の暗号化チャネルを確立する必要があります。

これには、ztunnel がノードローカルの Pod の各一意のアイデンティティ（サービスアカウント）ごとに異なるワークロード証明書を管理する必要があります。
ztunnel のアイデンティティは、ワークロード間の mTLS 接続には使用されません。

証明書を取得する際、ztunnel は自分のアイデンティティを CA に対して認証し、
別のワークロードのアイデンティティを要求します。
重要なのは、CA が ztunnel が該当のアイデンティティを要求する権限を持つことを強制する必要があることです。
ノード上で実行されていないアイデンティティに対するリクエストは拒否されます。
これは、感染したノードがメッシュ全体を危険に晒すことを防ぐために重要です。

この CA の強制実行は、Istio の CA が Kubernetes サービスアカウント JWT トークンを使用して行われます。
この JWT トークンは Pod 情報をエンコードします。
この強制実行は、ztunnel と統合された任意の代替 CA の要件でもあります。

ztunnel はノード上のすべてのアイデンティティに対して証明書を要求します。
これは、受信した{{< gloss "control plane" >}}コントロールプレーン{{< /gloss >}}の設定に基づいて決定されます。
ノード上で新しいアイデンティティが見つかった場合、それは低優先度で待ち行列に入れられ、最適化の一環として取得されます。
しかし、リクエストがまだ取得されていないアイデンティティを必要とする場合、そのアイデンティティはすぐに要求されます。

証明書がすぐに期限切れになる場合、ztunnel は証明書の更新を処理します。

##### 可観測性 {#telemetry}

ztunnel は [Istio 標準の TCP 指標](/ja/docs/reference/config/metrics/)をすべて出力します。

##### L4 トラフィックのデータプレーンの例 {#dataplane-example-for-layer-4-traffic}

L4 Ambient データプレーンは以下のようになります。

{{< image width="100%"
link="ztunnel-datapath-1.png"
caption="L4 データプレーンの ztunnel のみ"
>}}

この図は、Kubernetes クラスターのノード W1 と W2 上で実行される、Ambient メッシュに追加された複数のワークロードを示しています。
各ノードには ztunnel プロキシのインスタンスがあります。
このシナリオでは、アプリケーションクライアント Pod C1、C2、および C3 は Pod S1 が提供するサービスにアクセスする必要があります。
L7 トラフィックルーティングや L7 トラフィック管理などの高度な L7 機能は不要であるため、
L4 データプレーンで十分です。
{{< gloss "mutual tls authentication" >}}mTLS{{< /gloss >}} と L4 ポリシーの実行が可能であり、
waypoint プロキシは不要です。

以下の図は、ノード W1 上で実行される Pod C1 と C2 が、ノード W2 上で実行される Pod S1 に接続していることを示しています。

C1 と C2 の TCP トラフィックは、ztunnel が作成した {{< gloss >}}HBONE{{< /gloss >}} 接続を介して安全にトンネルされます。
{{< gloss "mutual tls authentication" >}}双向 TLS（mTLS）{{< /gloss >}} を使用して、トンネルトラフィックの双方向認証を行います。
[SPIFFE](https://github.com/spiffe/spiffe/blob/main/standards/SPIFFE.md) アイデンティティを使用して、接続の各端のワークロードを識別します。
トンネルプロトコルとトラフィックリダイレクトメカニズムの詳細については、[HBONE](/ja/docs/ambient/architecture/hbone) と [ztunnel トラフィックリダイレクト](/ja/docs/ambient/architecture/traffic-redirection)のガイドを参照してください。

{{< tip >}}
注意：図に表示されているように、HBONE トンネルは 2 つの ztunnel プロキシの間にありますが、
実際にはソース Pod と目的地 Pod の間にあります。
トラフィックはソース Pod 自体のネットワーク名前空間で HBONE でカプセル化および暗号化され、
最終的には目的地ワークノード上の目的地 Pod のネットワーク名前空間でカプセル化解除および復号化されます。
ztunnel プロキシは、論理的に HBONE トランスポートに必要なコントロールプレーンとデータプレーンを処理しますが、
それはソース Pod と目的地 Pod のネットワーク名前空間内部で実行できます。
{{< /tip >}}

ローカルトラフィック（図に示すように、ワークノード W2 上の Pod C3 から目的地 Pod S1）もローカル ztunnel プロキシインスタンスを通過するため、
L4 認証と L4 可観測性などの L4 トラフィック管理機能は、ノード境界をまたがっても同じ方法で実行されます。

## waypoint が有効なメッシュルーティング {#in-mesh-routing-with-waypoint-enabled}

Istio waypoint は専用で HBONE トラフィックを受信します。
リクエストを受信すると、waypoint はトラフィックが使用する `Pod` または `Service` に適用されることを確認します。

リクエストを受信すると、waypoint は L7 ポリシーを強制実行します
（例えば `AuthorizationPolicy`、`RequestAuthentication`、`WasmPlugin`、`Telemetry` など）。

`Pod` に直接送信されるリクエストは、ポリシーが適用された後に直接転送されます。

`Service` に送信されるリクエストについても、waypoint はルーティングと負荷分散を適用します。
デフォルトでは、`Service` はリクエストを自分自身にルーティングし、そのエンドポイント間で負荷分散を行います。
これは、`Service` に対するルーティングをオーバーライドすることができます。

例えば、以下のポリシーは `echo` サービスに対するリクエストが `echo-v1` に転送されることを確認します：

{{< text yaml >}}
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: echo
spec:
  parentRefs:
  - group: ""
    kind: Service
    name: echo
  rules:
  - backendRefs:
    - name: echo-v1
      port: 80
{{< /text >}}

以下の図は、ztunnel と waypoint の間のデータパス（L7 ポリシーが実行されている場合）を示しています。
ここでは、ztunnel は HBONE トンネルを使用してトラフィックを waypoint プロキシに送信して L7 処理を行います。
処理後、waypoint は 2 つ目の HBONE トンネルを使用してトラフィックを、選択されたサービスのターゲット Pod がホストされているノード上の ztunnel に送信します。
一般的に、waypoint プロキシはソースまたは目的地 Pod が配置されているノード上にある場合もあれば、そうでない場合もあります。

{{< image width="100%"
link="ztunnel-waypoint-datapath.png"
caption="ztunnel と waypoint の間のデータパス（L7 ポリシーが実行されている場合）"
>}}
