---
title: トラフィックルーティングの理解
linktitle: Traffic Routing
description: Istio がメッシュ内でどのようにトラフィックをルーティングするか。
weight: 30
keywords: [traffic-management, proxy]
owner: istio/wg-networking-maintainers
test: n/a
---

Istio の目標の 1 つは、既存のクラスターに「透過的プロキシ」として投入でき、従来通りにトラフィックが流れることです。
しかし、リクエストの負荷分散などの追加機能により、Istio は従来の Kubernetes クラスターとは異なり、より強力なトラフィック管理が可能です。
メッシュ内で何が起きているかを理解するには、Istio がどのようにトラフィックをルーティングするかを知ることが重要です。

{{< warning >}}
この記事は低レベルの実装詳細を説明しています。より高レベルの概要については、
トラフィック管理の[コンセプト](/ja/docs/concepts/traffic-management/)や[タスク](/ja/docs/tasks/traffic-management/)を参照してください。
{{< /warning >}}

## フロントエンドとバックエンド {#frontends-and-backends}

Istio のトラフィックルーティングには 2 つの主要な段階があります：

- 「フロントエンド」は、処理中のトラフィックタイプをどのようにマッチさせるかを指します。
  これは、どのバックエンドにトラフィックをルーティングし、どのポリシーを適用するかを決定するために必要です。
  たとえば、`http.ns.svc.cluster.local` の `Host` ヘッダーを読み取り、
  リクエストが `http` サービス宛であることを判断できます。
  このマッチングの仕組みについては後述します。

- 「バックエンド」は、トラフィックをマッチさせた後に送信する先を指します。
  上記の例では、リクエストが `http` サービス宛であると判断した場合、そのサービスのいずれかのエンドポイントに送信します。
  ただし、この選択は常に単純とは限らず、Istio では `VirtualService` のルールでこのロジックをカスタマイズできます。

標準の Kubernetes ネットワークにも同様の概念がありますが、より単純で通常は隠蔽されています。
`Service` を作成すると、通常は自動生成される DNS 名（例：`http.ns.svc.cluster.local`）に対応するフロントエンドと、
そのサービスを表す自動生成の IP アドレス（`ClusterIP`）が作成されます。
同様に、サービスが選択したすべての Pod を表すバックエンド（`Endpoints` または `EndpointSlice`）も作成されます。

## プロトコル {#protocols}

Kubernetes と異なり、Istio は HTTP や TLS などのアプリケーションレベルのプロトコルを扱えます。
これにより、Kubernetes で利用できるタイプよりも多様な[フロントエンド](#frontends-and-backends)のマッチングが可能です。

一般的に、Istio は 3 種類のプロトコルを理解します：

- HTTP（HTTP/1.1、HTTP/2、gRPC を含む。TLS で暗号化されたトラフィック（HTTPS）は含まれません）
- TLS（HTTPS を含む）
- 生の TCP バイトストリーム

[プロトコル選択](/ja/docs/ops/configuration/traffic-management/protocol-selection/)ドキュメントでは、
Istio がどのプロトコルを使うかをどのように決定するかを説明しています。

他の文脈では「TCP」という用語が UDP など他の L4 プロトコルと区別するために使われますが、
Istio での TCP プロトコルは通常「生のバイトストリーム」として扱われ、TLS や HTTP などのアプリケーションレベルのプロトコルは解釈されません。

## トラフィックルーティング {#traffic-routing}

Envoy プロキシがリクエストを受信すると、どこに転送するかを決定する必要があります。
デフォルトでは、リクエストは元のサービスに転送されますが、[カスタマイズ](/ja/docs/tasks/traffic-management/traffic-shifting/)も可能です。
処理方法はプロトコルによって異なります。

### TCP

TCP トラフィックを処理する際、Istio がルーティングに利用できる情報は非常に限られており（宛先 IP とポートのみ）、
これらの属性を使ってサービスを特定します。プロキシは各サービス IP（`<Kubernetes ClusterIP>:<Port>`）でリスンし、
トラフィックを上流サービスに転送します。

カスタマイズには TCP `VirtualService` を設定でき、
[特定の IP やポート](/ja/docs/reference/config/networking/virtual-service/#L4MatchAttributes)にマッチさせて、
リクエストとは異なる上流サービスにルーティングできます。

### TLS

TLS トラフィックを処理する際、Istio では生の TCP よりも多くの情報が利用できます。
TLS ハンドシェイク中に提示される [SNI](https://en.wikipedia.org/wiki/Server_Name_Indication) フィールドを参照できます。

標準サービスの場合は生の TCP と同様に IP:Port でマッチします。
ただし、Service IP が定義されていないサービス（[ExternalName サービス](#externalname-services)など）では、
SNI を使ってルーティングします。

また、TLS `VirtualService` でカスタムルーティングを設定し、
[SNI にマッチ](/ja/docs/reference/config/networking/virtual-service/#TLSMatchAttributes)してリクエストをカスタム宛先にルーティングできます。

### HTTP

HTTP では TCP や TLS よりも豊富なルーティングが可能です。HTTP では個々の HTTP リクエスト単位でルーティングでき、
ホスト、パス、ヘッダー、クエリパラメータなど多くの属性を利用できます。

TCP や TLS トラフィックは、Istio の有無にかかわらず（カスタムルーティングがなければ）同じ動作ですが、
HTTP では大きな違いがあります。

- Istio は個々のリクエスト単位で負荷分散を行います。これは特に長期接続（gRPC や HTTP/2 など）で有効で、
  コネクション単位の負荷分散が適さない場合に理想的です。

- リクエストはポートと **`Host` ヘッダー** でルーティングされます（ポートと IP ではありません）。
  つまり、宛先 IP アドレスは実際には無視されます。
  たとえば `curl 8.8.8.8 -H "Host: productpage.default.svc.cluster.local"`
  は `productpage` サービスにルーティングされます。

## マッチしないトラフィック {#unmatched-traffic}

上記のいずれの方法でもトラフィックがマッチしない場合、
[パススルー](/ja/docs/tasks/traffic-management/egress/egress-control/#envoy-passthrough-to-external-services)として扱われます。
デフォルトでは、これらのリクエストはそのまま転送され、Istio が認識しないサービス（`ServiceEntry` が作成されていない外部サービスなど）へのトラフィックも動作します。
ただし、これらのリクエスト転送時には双方向 TLS は使用されず、テレメトリー収集も制限されます。

## サービス種別 {#service-types}

標準の `ClusterIP` サービスに加え、Istio は Kubernetes の全サービス種別をサポートしています（いくつか注意点あり）。

### `LoadBalancer` および `NodePort` サービス {#loadbalancer-and-nodeport-services}

これらのサービスは `ClusterIP` サービスのスーパーセットであり、主に外部クライアントからのアクセスを許可するためのものです。
Istio はこれらのサービス種別をサポートしており、標準の `ClusterIP` サービスと同じ動作をします。

### ヘッドレスサービス {#headless-services}

[ヘッドレスサービス](https://kubernetes.io/ja/docs/concepts/services-networking/service/#headless-services)は `ClusterIP` を持たないサービスです。
代わりに、DNS 応答にはサービスに属する各エンドポイント（Pod IP）の IP アドレスが含まれます。

一般的に、Istio は各 Pod IP ごとにリスナーを設定しません（サービスレベルで動作するため）。
ただし、ヘッドレスサービスをサポートするため、ヘッドレスサービス内の各 IP:Port にリスナーを設定します。
例外として、プロトコルが HTTP の場合は `Host` ヘッダーでマッチします。

{{< warning >}}
Istio を使わない場合、ヘッドレスサービスの `ports` フィールドは厳密には必須ではありません。
リクエストは Pod IP に直接送信され、その IP はすべてのポートでトラフィックを受け入れられます。
しかし、Istio を使う場合はサービスでポートを宣言する必要があります。そうしないと[マッチしないトラフィック](/ja/docs/ops/configuration/traffic-management/traffic-routing/#unmatched-traffic)となります。
{{< /warning >}}

### ExternalName サービス {#externalname-services}

[ExternalName サービス](https://kubernetes.io/ja/docs/concepts/services-networking/service/#externalname)は本質的に DNS エイリアスです。

具体例として、次のようなケースを考えます：

{{< text yaml >}}
apiVersion: v1
kind: Service
metadata:
name: alias
spec:
type: ExternalName
externalName: concrete.example.com
{{< /text >}}

`ClusterIP` や Pod IP でマッチできないため、TCP トラフィックの場合、Istio でのトラフィックマッチングは変わりません。
Istio がリクエストを受信すると、`concrete.example.com` の IP を参照します。
これが Istio で認識されているサービスであれば、上述の通りルーティングされます。
そうでなければ、[マッチしないトラフィック](#unmatched-traffic)として扱われます。

HTTP や TLS のようにホスト名でマッチする場合は異なります。
宛先サービス（`concrete.example.com`）が Istio で認識されている場合、
ホストエイリアス（`alias.default.svc.cluster.local`）が [TLS](#tls) や [HTTP](#http) マッチの**追加**マッチ項目として追加されます。
そうでなければ、何も変わらず、[マッチしないトラフィック](#unmatched-traffic)として扱われます。

`ExternalName` サービス自体は[バックエンド](#frontends-and-backends)にはなれません。
既存サービスの追加[フロントエンド](#frontends-and-backends)マッチ項目としてのみ使えます。
`VirtualService` の宛先などで明示的にバックエンドとして指定した場合も同様です。
つまり、`alias.default.svc.cluster.local` を宛先に指定すると、リクエストは `concrete.example.com` に送信されます。
Istio がこのホスト名を認識していなければリクエストは失敗しますが、
この場合 `concrete.example.com` 用の `ServiceEntry` を作成すれば動作します。

### ServiceEntry

Kubernetes サービス以外にも、[ServiceEntry](/ja/docs/reference/config/networking/service-entry/#ServiceEntry) を作成して
Istio が認識するサービスセットを拡張できます。これにより、外部サービス（例：example.com）へのトラフィックにも Istio の機能を適用できます。

`addresses` を設定した ServiceEntry は、`ClusterIP` サービスと同じルーティングを行います。

ただし、`addresses` を持たない ServiceEntry は、そのポート上のすべての IP にマッチします。
これにより、同じポート上の[マッチしないトラフィック](#unmatched-traffic)が正しく転送されなくなる場合があります。
そのため、可能な限りこれらの使用は避けるか、必要な場合は専用ポートを使うのが望ましいです。
HTTP や TLS ではこの制約はなく、ホスト名/SNI ベースでルーティングされます。

{{< tip >}}
`addresses` フィールドと `endpoints` フィールドはよく混同されます。
`addresses` はマッチ対象の IP、`endpoints` はトラフィック送信先の IP 集合です。

たとえば、次の ServiceEntry は `1.1.1.1` へのトラフィックをマッチさせ、
設定した負荷分散ポリシーに従って `2.2.2.2` と `3.3.3.3` にリクエストを送信します：

{{< text yaml >}}
addresses: [1.1.1.1]
resolution: STATIC
endpoints:

- address: 2.2.2.2
- address: 3.3.3.3
  {{< /text  >}}

{{< /tip >}}
