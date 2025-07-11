---
title: Ambient と Kubernetes NetworkPolicy
description: CNI による L4 Kubernetes NetworkPolicy が Istio の Ambient モードとどのように連携するかを解説します。
weight: 20
owner: istio/wg-networking-maintainers
test: no
---

Kubernetes の [NetworkPolicy](https://kubernetes.io/ja/docs/concepts/services-networking/network-policies/) を使うと、L4 トラフィックが Pod に到達する方法を制御できます。

`NetworkPolicy` は通常、クラスターにインストールされた {{< gloss >}}CNI{{< /gloss >}} によって強制されます。
Istio は CNI ではなく、`NetworkPolicy` を強制したり管理したりしません。
すべての場合でそれに従い、Ambient も Kubernetes の `NetworkPolicy` の強制をバイパスすることはありません。

つまり、Kubernetes の `NetworkPolicy` を作成して Istio トラフィックをブロックしたり、Istio の機能を妨げたりすることができます。そのため、`NetworkPolicy` と Ambient を併用する場合はいくつか注意点があります。

## Ambient トラフィックカバレッジと Kubernetes NetworkPolicy {#ambient-traffic-overlay-and-kubernetes-networkpolicy}

アプリケーションが Ambient メッシュに追加されると、Ambient のセキュアな L4 カバレッジは Pod 間のトラフィックをポート 15008 経由で転送します。
セキュアトラフィックがターゲット Pod の 15008 ポートに到達すると、元のターゲットポートにプロキシされます。

ただし、`NetworkPolicy` は Pod の外側、ホスト上で強制されます。つまり、既存の `NetworkPolicy` で例えば Ambient Pod への 443 以外のすべてのポートへのインバウンドトラフィックを拒否している場合、ポート 15008 を例外として追加する必要があります。
トラフィックを受信する Sidecar ワークロードでも、Ambient ワークロードと通信するために 15008 ポートのインバウンドトラフィックを許可する必要があります。

例えば、以下の `NetworkPolicy` は、{{< gloss >}}HBONE{{< /gloss >}} の 15008 ポートへの `my-app` へのトラフィックをブロックします：

{{< text syntax=yaml snip_id=none >}}
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
name: my-app-allow-ingress-web
spec:
podSelector:
matchLabels:
app.kubernetes.io/name: my-app
ingress:

- ports: - port: 8080
  protocol: TCP
  {{< /text >}}

`my-app` を Ambient メッシュに追加した場合は、次のようにしてください：

{{< text syntax=yaml snip_id=none >}}
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
name: my-app-allow-ingress-web
spec:
podSelector:
matchLabels:
app.kubernetes.io/name: my-app
ingress:

- ports: - port: 8080
  protocol: TCP - port: 15008
  protocol: TCP
  {{< /text >}}

## Ambient、ヘルスプローブと Kubernetes NetworkPolicy {#ambient-health-probes-and-kubernetes-networkpolicy}

Kubernetes のヘルスチェックプローブには課題があり、Kubernetes のトラフィックポリシーに特別なケースを作り出します。
これらはクラスター内の他の Pod ではなく、ノード上のプロセスとして動作する kubelet から発信されます。
これらは平文かつ非セキュアです。kubelet や Kubernetes ノードは通常独自の暗号化アイデンティティを持たないため、アクセス制御ができません。
単にヘルスプローブ用ポートをすべて許可するだけでは不十分で、悪意のあるトラフィックも kubelet と同じようにそのポートを使えてしまいます。
また、多くのアプリケーションはヘルスプローブと通常のアプリトラフィックで同じポートを使うため、単純なポートベースの許可は適切ではありません。

CNI 実装ごとにこの問題への対処方法は異なり、kubelet のヘルスプローブを通常のポリシー強制から除外したり、例外を設けたりしています。

Istio Ambient では、iptables ルールと SNAT（ソースネットワークアドレス変換）を組み合わせて、ローカルノードの固定リンクローカル IP からのパケットのみを書き換え、Istio ポリシー実装がそれらを非セキュアなヘルスプローブトラフィックとして明示的に無視できるようにしています。
リンクローカル IP は、通常入口・出口制御で無視され、[IETF 標準](https://datatracker.ietf.org/doc/html/rfc3927)上もローカルサブネット外にはルーティングできないため、デフォルト IP として選ばれています。

Pod を Ambient メッシュに追加すると、この挙動はデフォルトで透過的に有効化され、Ambient はこのトラフィックの識別に `169.254.7.127` のリンクローカルアドレスを使います。

ただし、ワークロード・ネームスペース・クラスターに既存のインバウンドまたはアウトバウンド `NetworkPolicy` がある場合、CNI によってはこのリンクローカルアドレスのパケットが明示的な `NetworkPolicy` でブロックされ、Pod のヘルスプローブが Ambient メッシュ追加後に失敗することがあります。

例えば、以下の `NetworkPolicy` をネームスペースに適用すると、`my-app` Pod へのすべてのトラフィック（Istio も含む）がブロックされ、kubelet のヘルスプローブもブロックされます。CNI によっては kubelet プローブやリンクローカルアドレスがこのポリシーで無視される場合もあれば、ブロックされる場合もあります：

{{< text syntax=yaml snip_id=none >}}
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
name: deny-ingress
spec:
podSelector:
matchLabels:
app.kubernetes.io/name: my-app
policyTypes:

- Ingress
  {{< /text >}}

Pod が Ambient メッシュに登録されると、ヘルスプローブパケットは SNAT でリンクローカルアドレスが割り当てられるため、CNI の `NetworkPolicy` 実装によってはヘルスプローブがブロックされる場合があります。
Ambient のヘルスプローブが `NetworkPolicy` をバイパスできるようにするには、このトラフィックに使われるリンクローカルアドレスを許可リストに追加し、ホストノードから Pod へのトラフィックを明示的に許可してください：

{{< text syntax=yaml snip_id=none >}}
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
name: deny-ingress-allow-kubelet-healthprobes
spec:
podSelector:
matchLabels:
app.kubernetes.io/name: my-app
ingress: - from: - ipBlock:
cidr: 169.254.7.127/32
{{< /text >}}
