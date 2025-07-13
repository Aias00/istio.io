---
title: ワイルドカードホストの Egress
description: 汎用ドメイン内の一連のホストに対して、個別に設定することなく Egress を有効にする方法を説明します。
keywords: [traffic-management, egress]
weight: 50
aliases:
  - /zh/docs/examples/advanced-gateways/wildcard-egress-hosts/
owner: istio/wg-networking-maintainers
test: yes
---

[出口トラフィックの制御](/ja/docs/tasks/traffic-management/egress/)タスクや[egress ゲートウェイの構成](/ja/docs/tasks/traffic-management/egress/egress-gateway/)の例では、`edition.cnn.com` のような特定ホストの出口トラフィックを構成する方法を説明しました。本例では、`*.wikipedia.org` のような汎用ドメイン内の一連の特定ホストに対して、個別に設定することなく出口トラフィックを有効にする方法を説明します。

## 背景 {#background}

Istio で全言語の `wikipedia.org` サイトへの出口トラフィックを有効にしたいとします。各言語の `wikipedia.org` サイトはそれぞれ独自のホスト名（例：英語は `en.wikipedia.org`、ドイツ語は `de.wikipedia.org`）を持っています。すべての Wikipedia サイトへの出口トラフィックを、各言語ごとに個別設定せずに、汎用的な設定で有効にしたい場合にワイルドカードが役立ちます。

{{< boilerplate gateway-api-support >}}

## 始める前に {#before-you-begin}

- Istio をインストールし、アクセスログを有効化し、デフォルトで外部への出力トラフィックをブロックするポリシーを適用してください。

{{< tabset category-name="config-api" >}}

{{< tab name="Istio API" category-value="istio-apis" >}}

{{< text bash >}}
$ istioctl install --set profile=demo --set meshConfig.outboundTrafficPolicy.mode=REGISTRY_ONLY
{{< /text >}}

{{< tip >}}
`demo` プロファイル以外の Istio 設定でもこのタスクを実行できますが、[Istio Egress ゲートウェイのデプロイ](/ja/docs/tasks/traffic-management/egress/egress-gateway/#deploy-Istio-egress-gateway)、[Envoy のアクセスログ有効化](/ja/docs/tasks/observability/logs/access-log/#enable-envoy-s-access-logging)、[デフォルトで外部トラフィックをブロックするポリシーの適用](/ja/docs/tasks/traffic-management/egress/egress-control/#change-to-the-blocking-by-default-policy)を行ってください。
{{< /tip >}}

{{< /tab >}}

{{< tab name="Gateway API" category-value="gateway-api" >}}

{{< text bash >}}
$ istioctl install --set profile=minimal -y \
 --set values.pilot.env.PILOT_ENABLE_ALPHA_GATEWAY_API=true \
 --set meshConfig.accessLogFile=/dev/stdout \
 --set meshConfig.outboundTrafficPolicy.mode=REGISTRY_ONLY
{{< /text >}}

{{< /tab >}}

{{< /tabset >}}

- [curl]({{< github_tree >}}/samples/curl) サンプルアプリをデプロイし、リクエスト送信のテストソースとします。
  [Sidecar の自動注入](/ja/docs/setup/additional-setup/sidecar-injection/#automatic-sidecar-injection)が有効な場合は、次のコマンドでサンプルアプリをデプロイします：

  {{< text bash >}}
    $ kubectl apply -f @samples/curl/curl.yaml@
  {{< /text >}}

  そうでない場合は、`curl` アプリをデプロイする前に手動で Sidecar を注入してください：

  {{< text bash >}}
    $ kubectl apply -f <(istioctl kube-inject -f @samples/curl/curl.yaml@)
  {{< /text >}}

  {{< tip >}}
    任意の Pod で `curl` をテストソースとして利用できます。
  {{< /tip >}}

- `SOURCE_POD` 環境変数にテスト用 Pod 名を設定します：

  {{< text bash >}}
    $ export SOURCE_POD=$(kubectl get pod -l app=curl -o jsonpath={.items..metadata.name})
  {{< /text >}}

## ワイルドカードホストへのトラフィックを直接誘導する {#configure-direct-traffic-to-a-wildcard-host}

汎用ドメイン内の一連のホストにアクセスする最も簡単な方法は、ワイルドカードホストで単純な `ServiceEntry` を作成し、Sidecar から直接サービスを呼び出すことです。直接呼び出す場合（Egress ゲートウェイを経由しない場合）、ワイルドカードホストの設定は他のホスト（FQDN など）とほぼ同じですが、多数のホストをまとめて管理できる点が便利です。

{{< warning >}}
この設定は悪意のあるアプリケーションによって簡単に回避される可能性があります。セキュアな出口トラフィック制御を実現するには、Egress ゲートウェイ経由でトラフィックを誘導してください。
{{< /warning >}}

{{< warning >}}
ワイルドカードホストでは `DNS` 解決は利用できません。そのため、以下の ServiceEntry では `NONE` 解決（デフォルト）が使われています。
{{< /warning >}}

1. `*.wikipedia.org` 用の `ServiceEntry` を定義します：

   {{< text bash >}}
    $ kubectl apply -f - <<EOF
    apiVersion: networking.istio.io/v1
    kind: ServiceEntry
    metadata:
    name: wikipedia
    spec:
    hosts:

    - "\*.wikipedia.org"
      ports:
    - number: 443
      name: https
      protocol: HTTPS
      EOF
      {{< /text >}}

1. [https://en.wikipedia.org](https://en.wikipedia.org) および [https://de.wikipedia.org](https://de.wikipedia.org) へ HTTPS リクエストを送信します：

   {{< text bash >}}
    $ kubectl exec -it $SOURCE_POD -c curl -- sh -c 'curl -s https://en.wikipedia.org/wiki/Main_Page | grep -o "<title>._</title>"; curl -s https://de.wikipedia.org/wiki/Wikipedia:Hauptseite | grep -o "<title>._</title>"'
    <title>Wikipedia, the free encyclopedia</title>
    <title>Wikipedia – Die freie Enzyklopädie</title>
   {{< /text >}}

### ワイルドカードホストへの直接トラフィックルールのクリーンアップ {#cleanup-direct-traffic-to-a-wildcard-host}

{{< text bash >}}
$ kubectl delete serviceentry wikipedia
{{< /text >}}

## ワイルドカードホストへの Egress ゲートウェイルールの構成 {#configure-egress-gateway-traffic-to-a-wildcard-host}

すべてのワイルドカードホストに対して単一のサーバーがサービスを提供する場合、Egress ゲートウェイ経由でワイルドカードホストにアクセスする構成は通常のホストとほぼ同じです。ただし、ルーティング先はワイルドカードホストそのものではなく、汎用ドメイン集合の中の一意なサーバーホストに設定する必要があります。

1. _\*.wikipedia.org_ 用の Egress `Gateway` を作成し、Egress ゲートウェイ経由のトラフィックおよび Egress ゲートウェイから外部サービスへのトラフィックを誘導するルールを作成します。

{{< tabset category-name="config-api" >}}

{{< tab name="Istio API" category-value="istio-apis" >}}

{{< text bash >}}
$ kubectl apply -f - <<EOF
apiVersion: networking.istio.io/v1
kind: Gateway
metadata:
name: istio-egressgateway
spec:
selector:
istio: egressgateway
servers:

- port:
  number: 443
  name: https
  protocol: HTTPS
  hosts:
  - "\*.wikipedia.org"
    tls:
    mode: PASSTHROUGH

---

apiVersion: networking.istio.io/v1
kind: DestinationRule
metadata:
name: egressgateway-for-wikipedia
spec:
host: istio-egressgateway.istio-system.svc.cluster.local
subsets: - name: wikipedia

---

apiVersion: networking.istio.io/v1
kind: VirtualService
metadata:
name: direct-wikipedia-through-egress-gateway
spec:
hosts:

- "\*.wikipedia.org"
  gateways:
- mesh
- istio-egressgateway
  tls:
- match:
  - gateways:
    - mesh
      port: 443
      sniHosts:
    - "\*.wikipedia.org"
      route:
  - destination:
    host: istio-egressgateway.istio-system.svc.cluster.local
    subset: wikipedia
    port:
    number: 443
    weight: 100
- match: - gateways: - istio-egressgateway
  port: 443
  sniHosts: - "\*.wikipedia.org"
  route: - destination:
  host: www.wikipedia.org
  port:
  number: 443
  weight: 100
  EOF
  {{< /text >}}

{{< /tab >}}

{{< tab name="Gateway API" category-value="gateway-api" >}}

{{< text bash >}}
$ kubectl apply -f - <<EOF
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
name: wikipedia-egress-gateway
annotations:
networking.istio.io/service-type: ClusterIP
spec:
gatewayClassName: istio
listeners:

- name: tls
  hostname: "\*.wikipedia.org"
  port: 443
  protocol: TLS
  tls:
  mode: Passthrough
  allowedRoutes:
  namespaces:
  from: Same

---

apiVersion: gateway.networking.k8s.io/v1alpha2
kind: TLSRoute
metadata:
name: direct-wikipedia-to-egress-gateway
spec:
parentRefs:

- kind: ServiceEntry
  group: networking.istio.io
  name: wikipedia
  rules:
- backendRefs:
  - name: wikipedia-egress-gateway-istio
    port: 443

---

apiVersion: gateway.networking.k8s.io/v1alpha2
kind: TLSRoute
metadata:
name: forward-wikipedia-from-egress-gateway
spec:
parentRefs:

- name: wikipedia-egress-gateway
  hostnames:
- "\*.wikipedia.org"
  rules:
- backendRefs:
  - kind: Hostname
    group: networking.istio.io
    name: www.wikipedia.org
    port: 443

---

apiVersion: networking.istio.io/v1
kind: ServiceEntry
metadata:
name: wikipedia
spec:
hosts:

- "\*.wikipedia.org"
  ports:
- number: 443
  name: https
  protocol: HTTPS
  EOF
  {{< /text >}}

{{< /tab >}}

{{< /tabset >}}

2.  目的サーバー _www.wikipedia.org_ 用の `ServiceEntry` を作成します：

    {{< text bash >}}
    $ kubectl apply -f - <<EOF
    apiVersion: networking.istio.io/v1
    kind: ServiceEntry
    metadata:
    name: www-wikipedia
    spec:
    hosts:

    - www.wikipedia.org
      ports:
    - number: 443
      name: https
      protocol: HTTPS
      resolution: DNS
      EOF
      {{< /text >}}

3.  [https://en.wikipedia.org](https://en.wikipedia.org) および [https://de.wikipedia.org](https://de.wikipedia.org) へ HTTPS リクエストを送信します：

    {{< text bash >}}
    $ kubectl exec "$SOURCE_POD" -c curl -- sh -c 'curl -s https://en.wikipedia.org/wiki/Main_Page | grep -o "<title>._</title>"; curl -s https://de.wikipedia.org/wiki/Wikipedia:Hauptseite | grep -o "<title>._</title>"'
    <title>Wikipedia, the free encyclopedia</title>
    <title>Wikipedia – Die freie Enzyklopädie</title>
    {{< /text >}}

4.  Egress ゲートウェイプロキシの `*.wikipedia.org` へのアクセスカウンタ統計を確認します。

{{< tabset category-name="config-api" >}}

{{< tab name="Istio API" category-value="istio-apis" >}}

{{< text bash >}}
$ kubectl exec "$(kubectl get pod -l istio=egressgateway -n istio-system -o jsonpath='{.items[0].metadata.name}')" -c istio-proxy -n istio-system -- pilot-agent request GET clusters | grep '^outbound|443||www.wikipedia.org.*cx_total:'
outbound|443||www.wikipedia.org::208.80.154.224:443::cx_total::2
{{< /text >}}

{{< /tab >}}

{{< tab name="Gateway API" category-value="gateway-api" >}}

{{< text bash >}}
$ kubectl exec "$(kubectl get pod -l gateway.networking.k8s.io/gateway-name=wikipedia-egress-gateway -o jsonpath='{.items[0].metadata.name}')" -c istio-proxy -- pilot-agent request GET clusters | grep '^outbound|443||www.wikipedia.org.*cx_total:'
outbound|443||www.wikipedia.org::208.80.154.224:443::cx_total::2
{{< /text >}}

{{< /tab >}}

{{< /tabset >}}

### ワイルドカードホストへの Egress ゲートウェイルールのクリーンアップ {#cleanup-egress-gateway-traffic-to-a-wildcard-host}

{{< tabset category-name="config-api" >}}

{{< tab name="Istio API" category-value="istio-apis" >}}

{{< text bash >}}
$ kubectl delete serviceentry www-wikipedia
$ kubectl delete gateway istio-egressgateway
$ kubectl delete virtualservice direct-wikipedia-through-egress-gateway
$ kubectl delete destinationrule egressgateway-for-wikipedia
{{< /text >}}

{{< /tab >}}

{{< tab name="Gateway API" category-value="gateway-api" >}}

{{< text bash >}}
$ kubectl delete se wikipedia
$ kubectl delete se www-wikipedia
$ kubectl delete gtw wikipedia-egress-gateway
$ kubectl delete tlsroute direct-wikipedia-to-egress-gateway
$ kubectl delete tlsroute forward-wikipedia-from-egress-gateway
{{< /text >}}

{{< /tab >}}

{{< /tabset >}}

## 任意ドメインのワイルドカード構成 {#wildcard-configuration-for-arbitrary-domains}

前節の構成が有効なのは、`*.wikipedia.org` のすべてのサイトが任意の `wikipedia.org` サーバーでサービス提供される場合です。しかし、実際にはそうでない場合もあります。たとえば、より一般的なワイルドカードドメイン（`*.com` や `*.org` など）への出口制御を構成したい場合、Istio ゲートウェイでのルーティングは、事前定義されたホスト・IP アドレス、またはリクエストの元の宛先 IP アドレスにしか行えません。

前節では、仮想サービスでリクエストを事前定義ホスト `www.wikipedia.org` にルーティングしました。しかし、一般的なケースでは、リクエストで受け取った任意のホストにサービスを提供できるホストや IP アドレスが分からないため、リクエストの元の宛先アドレスだけが唯一のルーティング値となります。残念ながら、出口ゲートウェイを使う場合、元のリクエストはゲートウェイにリダイレクトされるため、元の宛先アドレスが失われ、宛先 IP アドレスはゲートウェイのものになります。

このような場合、[Envoy フィルタ](/ja/docs/reference/config/networking/envoy-filter/)を使い、[SNI](https://ja.wikipedia.org/wiki/Server_Name_Indication) を利用して、任意ドメインの HTTPS や TLS リクエストの値で元の宛先を識別しルーティングすることが可能です（ただし、Istio の実装詳細に依存するため簡単ではなく脆弱です）。この構成例は[出口トラフィックをワイルドカード宛先にルーティングする](/ja/blog/2023/egress-sni/)で紹介されています。

## クリーンアップ {#cleanup}

- [curl]({{< github_tree >}}/samples/curl) サービスを削除します：

  {{< text bash >}}
    $ kubectl delete -f @samples/curl/curl.yaml@
  {{< /text >}}

- クラスタから Istio をアンインストールします：

  {{< text bash >}}
    $ istioctl uninstall --purge -y
  {{< /text >}}
