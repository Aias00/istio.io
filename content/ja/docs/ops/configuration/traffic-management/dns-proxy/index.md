---
title: DNS プロキシ
description: DNS プロキシの設定方法。
weight: 60
keywords: [traffic-management, dns, virtual-machine]
owner: istio/wg-networking-maintainers
test: yes
---

Istio はアプリケーショントラフィックのキャプチャに加え、DNS リクエストもキャプチャして、
メッシュのパフォーマンスと可用性を向上させることができます。Istio が DNS をプロキシすると、
アプリケーションからのすべての DNS リクエストは Sidecar または ztunnel プロキシにリダイレクトされ、
Sidecar はドメイン名と IP アドレスのマッピングを保持します。リクエストがプロキシで処理される場合、
アプリケーションに直接応答を返し、上流の DNS サーバーへの往復を回避します。
そうでない場合、リクエストは標準の `/etc/resolv.conf` DNS 設定に従って上流に転送されます。

Kubernetes は Kubernetes の `Service` に対して標準の DNS 解決を提供しますが、
カスタムの `ServiceEntry` は認識されません。この機能により、`ServiceEntry` のアドレスも解決でき、
カスタム DNS サービス設定は不要です。Kubernetes `Service` についても、
同じ DNS 応答が得られますが、`kube-dns` の負荷が減り、パフォーマンスが向上します。

この機能は Kubernetes 外部で稼働するサービスにも適用できます。
つまり、すべての内部サービスが解決可能となり、
クラスター外の Kubernetes DNS エントリを公開するための煩雑な方法が不要になります。

## はじめに {#getting-started}

Istio は通常、HTTP ヘッダーに基づいてトラフィックをルーティングします。HTTP ヘッダーに基づくルーティングができない場合
（たとえば Ambient モードや、Sidecar モードで TCP トラフィックを使用する場合）、DNS プロキシを有効にできます。

Ambient モードでは、ztunnel はレイヤー 4 のトラフィックしか見えず、HTTP ヘッダーにアクセスできません。
そのため、DNS プロキシメカニズムは `ServiceEntry` アドレスの解決に必須であり、
特に[外部トラフィックを waypoint に送信する場合](https://github.com/istio/istio/wiki/Troubleshooting-Istio-Ambient#scenario-ztunnel-is-not-sending-egress-traffic-to-waypoints)に重要です。

### Ambient モード {#ambient-mode}

Istio 1.25 以降、Ambient モードでは DNS プロキシメカニズムがデフォルトで有効になっています。

1.25 より前のバージョンでは、インストール時に `values.cni.ambient.dnsCapture=true`
および `values.pilot.env.PILOT_ENABLE_IP_AUTOALLOCATE=true` を設定することで DNS キャプチャを有効にできます。

### Sidecar モード {#sidecar-mode}

この機能はデフォルトでは無効です。有効にするには、Istio インストール時に以下の設定を使用します：

{{< text bash >}}
$ cat <<EOF | istioctl install -y -f -
apiVersion: install.istio.io/v1alpha1
kind: IstioOperator
spec:
meshConfig:
defaultConfig:
proxyMetadata: # 基本的な DNS プロキシを有効化
ISTIO_META_DNS_CAPTURE: "true"
EOF
{{< /text >}}

また、各 Pod ごとに [`proxy.istio.io/config` アノテーション](/ja/docs/reference/config/annotations/)で有効化することもできます：

{{< text syntax=yaml snip_id=none >}}
kind: Deployment
metadata:
  name: curl
spec:
...
  template:
    metadata:
      annotations:
        proxy.istio.io/config: |
          proxyMetadata:
            ISTIO_META_DNS_CAPTURE: "true"
...
{{< /text >}}

{{< tip >}}
[`istioctl ワークロード構成`](/ja/docs/setup/install/virtual-machine/)で仮想マシンをデプロイする場合、
デフォルトで基本的な DNS プロキシが有効になります。
{{< /tip >}}

## DNS キャプチャの動作例 {#DNS-capture-in-action}

DNS キャプチャを試すには、まずいくつかの外部サービス用に `ServiceEntry` を作成します：

{{< text bash >}}
$ kubectl apply -f - <<EOF
apiVersion: networking.istio.io/v1
kind: ServiceEntry
metadata:
name: external-address
spec:
addresses:

- 198.51.100.1
  hosts:
- address.internal
  ports:
- name: http
  number: 80
  protocol: HTTP
  EOF
  {{< /text >}}

クライアントアプリから DNS リクエストを発行します：

{{< text bash >}}
$ kubectl label namespace default istio-injection=enabled --overwrite
$ kubectl apply -f @samples/curl/curl.yaml@
{{< /text >}}

DNS キャプチャを有効にしない場合、`address.internal` へのリクエストは解決できない可能性があります。
DNS キャプチャを有効にすると、`address` 設定に基づいた応答が返されます：

{{< text bash >}}
$ kubectl exec deploy/curl -- curl -sS -v address.internal

- Trying 198.51.100.1:80...
  {{< /text >}}

## アドレスの自動割り当て {#address-auto-allocation}

上記の例では、リクエスト先サービスに事前定義された IP アドレスがあります。
しかし、一般的なケースでは、外部サービスへのアクセス時に固定アドレスがないため、
DNS プロキシを通じて外部サービスにアクセスする必要があります。DNS プロキシが十分な情報を持たない場合、
上流に DNS リクエストを転送する必要があります。

これは TCP 通信では大きな問題となります。HTTP リクエストのように `Host` ヘッダーでルーティングできず、
TCP では宛先 IP とポート番号のみでルーティングします。
バックエンドに安定した IP がないため、他の情報でルーティングできず、
ポート番号だけが頼りですが、これでは複数の `ServiceEntry` で同じ TCP ポートを共有すると競合が発生します。
詳細は[以下のセクション](#external-tcp-services-without-vips)を参照してください。

この問題を解決するため、DNS プロキシは明示的にアドレスが指定されていない `ServiceEntry` に対して、
自動的にアドレスを割り当てる機能もサポートしています。
DNS 応答には各 `ServiceEntry` ごとに一意で自動割り当てされたアドレスが含まれます。
プロキシはこのアドレスにリクエストをマッチさせ、対応する `ServiceEntry` に転送します。
ワイルドカードホストを使わない限り、Istio はこれらのサービスにルーティング不可能な仮想 IP（Class E サブネットから）を自動割り当てします。
Sidecar 上の Istio プロキシは、アプリケーションの DNS クエリに対してこれらの仮想 IP を返します。
Envoy はこれで各外部 TCP サービスのトラフィックを明確に区別し、正しい宛先に転送できます。

{{< warning >}}
この機能は DNS 応答を変更するため、すべてのアプリケーションと互換性があるとは限りません。
{{< /warning >}}

別の `ServiceEntry` を設定してみましょう：

{{< text bash >}}
$ kubectl apply -f - <<EOF
apiVersion: networking.istio.io/v1
kind: ServiceEntry
metadata:
name: external-auto
spec:
hosts:

- auto.internal
  ports:
- name: http
  number: 80
  protocol: HTTP
  resolution: DNS
  EOF
  {{< /text >}}

次にリクエストを送信します：

{{< text bash >}}
$ kubectl exec deploy/curl -- curl -sS -v auto.internal

- Trying 240.240.0.1:80...
  {{< /text >}}

このように、リクエストは自動割り当てされたアドレス `240.240.0.1` に送信されます。
これらのアドレスは `240.240.0.0/16` の予約済み IP アドレスプールから選ばれ、
実際のサービスとの競合を避けます。

ユーザーは `ServiceEntry` に
`networking.istio.io/enable-autoallocate-ip="true/false"`
というラベルを追加することで、より細かく制御できます。このラベルは `spec.addresses` を設定していない
`ServiceEntry` に対して自動割り当てを有効または無効にします。

この機能を試すには、既存の `ServiceEntry` をラベル付きで更新してください：

{{< text bash >}}
$ kubectl apply -f - <<EOF
apiVersion: networking.istio.io/v1
kind: ServiceEntry
metadata:
name: external-auto
labels:
networking.istio.io/enable-autoallocate-ip: "false"
spec:
hosts:

- auto.internal
  ports:
- name: http
  number: 80
  protocol: HTTP
  resolution: DNS
  EOF
  {{< /text >}}

今度はリクエストを送信し、自動割り当てが行われないことを確認します：

{{< text bash >}}
$ kubectl exec deploy/curl -- curl -sS -v auto.internal

- Could not resolve host: auto.internal
- shutting down connection #0
  {{< /text >}}

## VIP を持たない外部 TCP サービス {#external-tcp-services-without-vips}

デフォルトでは、Istio は外部 TCP トラフィックのルーティングに制限があります。
同じポート上の複数の TCP サービスを区別できないためです。
サードパーティのデータベース（AWS RDS など）や地理的冗長構成のデータベースでは、
この制限が特に顕著です。
デフォルトでは、似ているが異なる外部 TCP サービスを個別に扱うことはできません。
Sidecar がメッシュ外の 2 つの異なる TCP サービスのトラフィックを区別するには、
それぞれ異なるポートを使うか、グローバルに一意な VIP アドレスが必要です。

たとえば、2 つの外部データベースサービス（`mysql-instance1` と `mysql-instance2`）があり、
それぞれにサービスエントリを作成した場合でも、クライアント Sidecar では `0.0.0.0:{port}`
に単一のリスナーしか持てません。パブリック DNS サーバーから `mysql-instance1` の IP だけを取得し、
トラフィックをそちらに転送します。`mysql-instance2` へのトラフィックは区別できません。
`0.0.0.0:{port}` に届くトラフィックがどちらのサービス宛か判別できないためです。

以下の例は、DNS プロキシを使ってこの問題を解決する方法を示します。
仮想 IP アドレスが各サービスエントリに割り当てられ、クライアント Sidecar は各外部 TCP サービスのトラフィックを明確に区別できます。

1.  [はじめに](#getting-started)セクションの Istio 設定を更新し、
    `discoverySelectors` を設定して、`istio-injection` が有効なネームスペースだけをメッシュの対象にします。
    これにより、クラスター内の他のネームスペースでメッシュ外の TCP サービスを実行できます。

    {{< text bash >}}
    $ cat <<EOF | istioctl install -y -f -
    apiVersion: install.istio.io/v1alpha1
    kind: IstioOperator
    spec:
    meshConfig:
    defaultConfig:
    proxyMetadata: # 基本的な DNS プロキシを有効化
    ISTIO_META_DNS_CAPTURE: "true" # 以下の discoverySelectors 設定は外部サービス TCP シナリオのシミュレーション用です。 # 外部サイトを使わずにテストできます。
    discoverySelectors: - matchLabels:
    istio-injection: enabled
    EOF
    {{< /text >}}

1.  1 つ目の外部サンプル TCP アプリをデプロイ：

    {{< text bash >}}
    $ kubectl create ns external-1
    $ kubectl -n external-1 apply -f samples/tcp-echo/tcp-echo.yaml
    {{< /text >}}

1.  2 つ目の外部サンプル TCP アプリをデプロイ：

    {{< text bash >}}
    $ kubectl create ns external-2
    $ kubectl -n external-2 apply -f samples/tcp-echo/tcp-echo.yaml
    {{< /text >}}

1.  外部サービスへの `ServiceEntry` を設定：

    {{< text bash >}}
    $ kubectl apply -f - <<EOF
    apiVersion: networking.istio.io/v1
    kind: ServiceEntry
    metadata:
    name: external-svc-1
    spec:
    hosts:

    - tcp-echo.external-1.svc.cluster.local
      ports:
    - name: external-svc-1
      number: 9000
      protocol: TCP
      resolution: DNS

    ***

    apiVersion: networking.istio.io/v1
    kind: ServiceEntry
    metadata:
    name: external-svc-2
    spec:
    hosts:

    - tcp-echo.external-2.svc.cluster.local
      ports:
    - name: external-svc-2
      number: 9000
      protocol: TCP
      resolution: DNS
      EOF
      {{< /text >}}

1.  クライアント側で各サービスごとにリスナーが設定されていることを確認：

    {{< text bash >}}
    $ istioctl pc listener deploy/curl | grep tcp-echo | awk '{printf "ADDRESS=%s, DESTINATION=%s %s\n", $1, $4, $5}'
    ADDRESS=240.240.105.94, DESTINATION=Cluster: outbound|9000||tcp-echo.external-2.svc.cluster.local
    ADDRESS=240.240.69.138, DESTINATION=Cluster: outbound|9000||tcp-echo.external-1.svc.cluster.local
    {{< /text >}}

## クリーンアップ {#cleanup}

{{< text bash >}}
$ kubectl -n external-1 delete -f @samples/tcp-echo/tcp-echo.yaml@
$ kubectl -n external-2 delete -f @samples/tcp-echo/tcp-echo.yaml@
$ kubectl delete -f @samples/curl/curl.yaml@
$ istioctl uninstall --purge -y
$ kubectl delete ns istio-system external-1 external-2
$ kubectl label namespace default istio-injection-
{{< /text >}}
