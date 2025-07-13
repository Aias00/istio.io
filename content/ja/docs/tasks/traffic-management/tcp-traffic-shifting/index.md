---
title: TCP トラフィックシフト
description: サービスの TCP トラフィックを旧バージョンから新バージョンへ移行する方法を紹介します。
weight: 31
keywords: [traffic-management, tcp-traffic-shifting]
aliases:
  - /zh/docs/tasks/traffic-management/tcp-version-migration.html
owner: istio/wg-networking-maintainers
test: yes
---

このタスクでは、マイクロサービスの TCP トラフィックをあるバージョンから別のバージョンへ移行する方法を紹介します。

よくあるユースケースは、マイクロサービスの旧バージョンから新バージョンへ TCP トラフィックを段階的に移行することです。
Istio では、一連のルーティングルールを設定することで、TCP トラフィックの一定割合をある宛先から別の宛先へリダイレクトできます。

このタスクでは、まず TCP トラフィックの 100% を `tcp-echo:v1` に割り当てます。
次に、Istio のルーティング重み付けを使って TCP トラフィックの 20% を `tcp-echo:v2` に割り当てます。

{{< boilerplate gateway-api-gamma-experimental >}}

## 始める前に {#before-you-begin}

- [インストールガイド](/ja/docs/setup/) の手順に従って Istio をインストールします。

- [トラフィック管理](/ja/docs/concepts/traffic-management) の概念ドキュメントを確認してください。

## テスト環境のセットアップ {#set-up-the-test-environment}

1.  まず、TCP トラフィックシフトのテスト用に名前空間を作成します。

    {{< text bash >}}
    $ kubectl create namespace istio-io-tcp-traffic-shifting
    {{< /text >}}

1.  [curl]({{< github_tree >}}/samples/curl) サンプルアプリケーションをデプロイし、リクエスト送信元のテスト用とします。

    {{< text bash >}}
    $ kubectl apply -f @samples/curl/curl.yaml@ -n istio-io-tcp-traffic-shifting
    {{< /text >}}

1.  `tcp-echo` マイクロサービスの `v1` と `v2` バージョンをデプロイします。

    {{< text bash >}}
    $ kubectl apply -f @samples/tcp-echo/tcp-echo-services.yaml@ -n istio-io-tcp-traffic-shifting
    {{< /text >}}

## 重み付けによる TCP ルーティングの適用 {#apply-weight-based-TCP-routing}

1. すべての TCP トラフィックを `tcp-echo` サービスの `v1` バージョンにルーティングします。

{{< tabset category-name="config-api" >}}

{{< tab name="Istio APIs" category-value="istio-apis" >}}

{{< text bash >}}
$ kubectl apply -f @samples/tcp-echo/tcp-echo-all-v1.yaml@ -n istio-io-tcp-traffic-shifting
{{< /text >}}

{{< /tab >}}

{{< tab name="Gateway API" category-value="gateway-api" >}}

{{< text bash >}}
$ kubectl apply -f @samples/tcp-echo/gateway-api/tcp-echo-all-v1.yaml@ -n istio-io-tcp-traffic-shifting
{{< /text >}}

{{< /tab >}}

{{< /tabset >}}

2. Ingress IP とポートを確認します：

{{< tabset category-name="config-api" >}}

{{< tab name="Istio APIs" category-value="istio-apis" >}}

[Ingress IP とポートの確認](/ja/docs/tasks/traffic-management/ingress/ingress-control/#determining-the-ingress-ip-and-ports) の手順に従い、
`TCP_INGRESS_PORT` と `INGRESS_HOST` 環境変数を設定します。

{{< /tab >}}

{{< tab name="Gateway API" category-value="gateway-api" >}}

以下のコマンドで `SECURE_INGRESS_PORT` と `INGRESS_HOST` 環境変数を設定します：

{{< text bash >}}
$ kubectl wait --for=condition=programmed gtw tcp-echo-gateway -n istio-io-tcp-traffic-shifting
$ export INGRESS_HOST=$(kubectl get gtw tcp-echo-gateway -n istio-io-tcp-traffic-shifting -o jsonpath='{.status.addresses[0].value}')
$ export TCP_INGRESS_PORT=$(kubectl get gtw tcp-echo-gateway -n istio-io-tcp-traffic-shifting -o jsonpath='{.spec.listeners[?(@.name=="tcp-31400")].port}')
{{< /text >}}

{{< /tab >}}

{{< /tabset >}}

3. いくつかの TCP トラフィックを送信して `tcp-echo` サービスが起動していることを確認します。

   {{< text bash >}}
    $ export CURL=$(kubectl get pod -l app=curl -n istio-io-tcp-traffic-shifting -o jsonpath={.items..metadata.name})
    $ for i in {1..20}; do \
    kubectl exec "$CURL" -c curl -n istio-io-tcp-traffic-shifting -- sh -c "(date; curl 1) | nc $INGRESS_HOST $TCP_INGRESS_PORT"; \
    done
    one Mon Nov 12 23:24:57 UTC 2022
    one Mon Nov 12 23:25:00 UTC 2022
    one Mon Nov 12 23:25:02 UTC 2022
    one Mon Nov 12 23:25:05 UTC 2022
    one Mon Nov 12 23:25:07 UTC 2022
    one Mon Nov 12 23:25:10 UTC 2022
    one Mon Nov 12 23:25:12 UTC 2022
    one Mon Nov 12 23:25:15 UTC 2022
    one Mon Nov 12 23:25:17 UTC 2022
    one Mon Nov 12 23:25:19 UTC 2022
    ...
   {{< /text >}}

   すべてのタイムスタンプに「**one**」というプレフィックスが付いていることに注目してください。これは、すべてのトラフィックが `tcp-echo` サービスの `v1` バージョンにルーティングされていることを示しています。

4. 次のコマンドで、`tcp-echo:v1` から `tcp-echo:v2` へ 20% のトラフィックを移行します：

{{< tabset category-name="config-api" >}}

{{< tab name="Istio APIs" category-value="istio-apis" >}}

{{< text bash >}}
$ kubectl apply -f @samples/tcp-echo/tcp-echo-20-v2.yaml@ -n istio-io-tcp-traffic-shifting
{{< /text >}}

{{< /tab >}}

{{< tab name="Gateway API" category-value="gateway-api" >}}

{{< text bash >}}
$ kubectl apply -f @samples/tcp-echo/gateway-api/tcp-echo-20-v2.yaml@ -n istio-io-tcp-traffic-shifting
{{< /text >}}

{{< /tab >}}

{{< /tabset >}}

5. 数秒待って新しいルールが伝播されたことを確認し、ルールが置き換わったことを確認します：

{{< tabset category-name="config-api" >}}

{{< tab name="Istio APIs" category-value="istio-apis" >}}

{{< text bash yaml >}}
$ kubectl get virtualservice tcp-echo -o yaml -n istio-io-tcp-traffic-shifting
apiVersion: networking.istio.io/v1
kind: VirtualService
...
spec:
...
tcp:

- match: - port: 31400
  route: - destination:
  host: tcp-echo
  port:
  number: 9000
  subset: v1
  weight: 80 - destination:
  host: tcp-echo
  port:
  number: 9000
  subset: v2
  weight: 20
  {{< /text >}}

{{< /tab >}}

{{< tab name="Gateway API" category-value="gateway-api" >}}

{{< text bash >}}
$ kubectl get tcproute tcp-echo -o yaml -n istio-io-tcp-traffic-shifting
apiVersion: gateway.networking.k8s.io/v1alpha2
kind: TCPRoute
...
spec:
parentRefs:

- group: gateway.networking.k8s.io
  kind: Gateway
  name: tcp-echo-gateway
  sectionName: tcp-31400
  rules:
- backendRefs: - group: ""
  kind: Service
  name: tcp-echo-v1
  port: 9000
  weight: 80 - group: ""
  kind: Service
  name: tcp-echo-v2
  port: 9000
  weight: 20
  ...
  {{< /text >}}

{{< /tab >}}

{{< /tabset >}}

6. さらに TCP トラフィックを `tcp-echo` マイクロサービスに送信します。

   {{< text bash >}}
    $ export CURL=$(kubectl get pod -l app=curl -n istio-io-tcp-traffic-shifting -o jsonpath={.items..metadata.name})
    $ for i in {1..20}; do \
    kubectl exec "$CURL" -c curl -n istio-io-tcp-traffic-shifting -- sh -c "(date; curl 1) | nc $INGRESS_HOST $TCP_INGRESS_PORT"; \
    done
    one Mon Nov 12 23:38:45 UTC 2022
    two Mon Nov 12 23:38:47 UTC 2022
    one Mon Nov 12 23:38:50 UTC 2022
    one Mon Nov 12 23:38:52 UTC 2022
    one Mon Nov 12 23:38:55 UTC 2022
    two Mon Nov 12 23:38:57 UTC 2022
    one Mon Nov 12 23:39:00 UTC 2022
    one Mon Nov 12 23:39:02 UTC 2022
    one Mon Nov 12 23:39:05 UTC 2022
    one Mon Nov 12 23:39:07 UTC 2022
    ...
   {{< /text >}}

   約 20% のタイムスタンプに「**two**」というプレフィックスが付いていることに注目してください。これは、TCP トラフィックの 80% が `tcp-echo` サービスの `v1` バージョンに、20% が `v2` バージョンにルーティングされていることを示しています。

## 仕組みの理解 {#understanding-what-happened}

このタスクでは、Istio のルーティング重み付け機能を使って `tcp-echo` サービスの TCP トラフィックを旧バージョンから新バージョンへ移行しました。これは、コンテナオーケストレーションプラットフォームのデプロイ機能によるバージョン移行とは異なります。後者（コンテナオーケストレーションプラットフォーム）はインスタンスのスケールアウトによってトラフィックを管理します。

Istio では、`tcp-echo` サービスの 2 つのバージョンを個別にスケールイン・スケールアウトでき、この操作は両バージョン間のトラフィック分配に影響しません。

異なるバージョン間のトラフィック管理や自動スケーリングの詳細については、[Istio でのカナリアリリース](/ja/blog/2017/0.1-canary/) のブログ記事をご覧ください。

## クリーンアップ {#cleanup}

1. ルーティングルールを削除します：

{{< tabset category-name="config-api" >}}

{{< tab name="Istio APIs" category-value="istio-apis" >}}

{{< text bash >}}
$ kubectl delete -f @samples/tcp-echo/tcp-echo-all-v1.yaml@ -n istio-io-tcp-traffic-shifting
{{< /text >}}

{{< /tab >}}

{{< tab name="Gateway API" category-value="gateway-api" >}}

{{< text bash >}}
$ kubectl delete -f @samples/tcp-echo/gateway-api/tcp-echo-all-v1.yaml@ -n istio-io-tcp-traffic-shifting
{{< /text >}}

{{< /tab >}}

{{< /tabset >}}

2. `curl` サンプル、`tcp-echo` アプリケーション、テスト用名前空間を削除します：

   {{< text bash >}}
    $ kubectl delete -f @samples/curl/curl.yaml@ -n istio-io-tcp-traffic-shifting
    $ kubectl delete -f @samples/tcp-echo/tcp-echo-services.yaml@ -n istio-io-tcp-traffic-shifting
    $ kubectl delete namespace istio-io-tcp-traffic-shifting
   {{< /text >}}
