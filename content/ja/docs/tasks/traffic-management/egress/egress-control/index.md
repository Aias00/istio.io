---
title: 外部サービスへのアクセス
description: Istio でサービスメッシュ内のサービスから外部サービスへのトラフィックをルーティングする方法を説明します。
weight: 10
aliases:
  - /zh/docs/tasks/egress.html
  - /zh/docs/tasks/egress
keywords: [traffic-management, egress]
owner: istio/wg-networking-maintainers
test: yes
---

Istio 対応 Pod からのすべての送信トラフィックはデフォルトで Sidecar プロキシにリダイレクトされるため、クラスタ外部の URL へのアクセス可否はプロキシの設定に依存します。デフォルトでは、Istio は Envoy プロキシを未知のサービスへのリクエストをパススルーするように設定します。これは Istio の導入を容易にしますが、通常はより厳格な制御を設定することが推奨されます。

このタスクでは、外部サービスへのアクセス方法を 3 つ紹介します：

1. Envoy プロキシがメッシュ内で設定されていないサービスへのリクエストをパススルーする方法。
1. [Service Entry](/ja/docs/reference/config/networking/service-entry/) を使って外部サービスへの制御されたアクセスを提供する方法。
1. 特定の IP 範囲については Envoy プロキシを完全にバイパスする方法。

## 始める前に {#before-you-begin}

- [インストールガイド](/ja/docs/setup/) の手順に従って Istio をセットアップしてください。
  `demo` [インストールプロファイル](/ja/docs/setup/additional-setup/config-profiles/) または
  [Envoy のアクセスログを有効化](/ja/docs/tasks/observability/logs/access-log/#enable-envoy-s-access-logging) してください。

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
  `curl` がインストールされている任意の Pod をテストソースとして利用できます。
  {{< /tip >}}

- 環境変数 `SOURCE_POD` を、テストソース Pod の名前で設定します：

  {{< text bash >}}
  $ export SOURCE_POD=$(kubectl get pod -l app=curl -o jsonpath='{.items..metadata.name}')
  {{< /text >}}

## Envoy で外部サービスへパススルー {#envoy-passthrough-to-external-services}

Istio には [インストールオプション](/ja/docs/reference/config/installation-options/) `global.outboundTrafficPolicy.mode` があり、Sidecar が外部サービス（Istio の内部サービスレジストリに定義されていないサービス）をどう扱うかを設定できます。`ALLOW_ANY` に設定すると、Istio プロキシは未知のサービスへの呼び出しを許可します。`REGISTRY_ONLY` に設定すると、Istio プロキシはメッシュ内で定義されていない HTTP サービスや Service Entry のホストへのアクセスをブロックします。`ALLOW_ANY` がデフォルト値で、外部サービスへのアクセス制御は行われません。

1. この方法の動作を確認するには、Istio のインストール時に `meshConfig.outboundTrafficPolicy.mode` オプションが `ALLOW_ANY` になっていることを確認してください。デフォルトで有効ですが、インストール時に明示的に `REGISTRY_ONLY` にしていなければ `ALLOW_ANY` です。

   設定を確認するには、次のコマンドを実行します：

   {{< text bash >}}
   $ kubectl get configmap istio -n istio-system -o yaml
   {{< /text >}}

   `meshConfig.outboundTrafficPolicy.mode` が明示的に `REGISTRY_ONLY` でなければ、`ALLOW_ANY` です。

   {{< tip >}}
   もし `REGISTRY_ONLY` モードにしていた場合は、次のように `ALLOW_ANY` に戻せます：

   {{< text syntax=bash snip_id=none >}}
   $ istioctl install <flags-you-used-to-install-Istio> --set meshConfig.outboundTrafficPolicy.mode=ALLOW_ANY
   {{< /text >}}

   {{< /tip >}}

1. `SOURCE_POD` から外部 HTTPS サービスに 2 回リクエストし、HTTP ステータス 200 が返ることを確認します：

   {{< text bash >}}
   $ kubectl exec "$SOURCE_POD" -c curl -- curl -sSI https://www.google.com | grep  "HTTP/"; kubectl exec "$SOURCE_POD" -c curl -- curl -sI https://edition.cnn.com | grep "HTTP/"
   HTTP/2 200
   HTTP/2 200
   {{< /text >}}

おめでとうございます！メッシュから外部へのトラフィック送信に成功しました。

この方法は簡単ですが、外部サービスへのトラフィックに対する Istio の監視や制御ができません。
次のセクションでは、外部サービスへのアクセスを監視・制御する方法を紹介します。

## 外部サービスへのアクセス制御 {#controlled-access-to-external-services}

Istio の `ServiceEntry` を使うことで、Istio クラスタから任意の公開サービスにアクセスできます。
ここでは、Istio のトラフィック監視や制御機能を失わずに、外部 HTTP サービス（[httpbin.org](http://httpbin.org)）や外部 HTTPS サービス（[www.google.com](https://www.google.com)）へのアクセスを設定する方法を紹介します。

### デフォルトでブロックするポリシーへの変更 {#change-to-the-blocking-by-default-policy}

外部サービスへのアクセス制御を示すため、`global.outboundTrafficPolicy.mode` オプションを `ALLOW_ANY` から `REGISTRY_ONLY` に変更します。

{{< tip >}}
`ALLOW_ANY` モードのままでも、特定の外部サービスにアクセス制御を追加できます。
この方法で、一部の外部サービスにのみ Istio の機能を適用し、他のサービスは制限しません。
すべてのサービスの設定が完了したら、モードを `REGISTRY_ONLY` に切り替えて意図しないアクセスをブロックできます。
{{< /tip >}}

1. `global.outboundTrafficPolicy.mode` オプションを `REGISTRY_ONLY` に変更します：

   `IstioOperator` 設定でインストールしている場合は、次のフィールドを追加します：

   {{< text yaml >}}
   spec:
   meshConfig:
   outboundTrafficPolicy:
   mode: REGISTRY_ONLY
   {{< /text >}}

   それ以外の場合は、`istioctl install` コマンドに次のオプションを追加します：

   {{< text syntax=bash snip_id=none >}}
   $ istioctl install <flags-you-used-to-install-Istio> \
    --set meshConfig.outboundTrafficPolicy.mode=REGISTRY_ONLY
   {{< /text >}}

1. `SOURCE_POD` から外部 HTTPS サービスにリクエストし、ブロックされることを確認します：

   {{< text bash >}}
   $ kubectl exec "$SOURCE_POD" -c curl -- curl -sI https://www.google.com | grep  "HTTP/"; kubectl exec "$SOURCE_POD" -c curl -- curl -sI https://edition.cnn.com | grep "HTTP/"
   command terminated with exit code 35
   command terminated with exit code 35
   {{< /text >}}

   {{< warning >}}
   設定変更が反映されるまで少し時間がかかる場合があります。数秒待ってから再度コマンドを実行してください。
   {{< /warning >}}

### 外部 HTTP サービスへのアクセス {#access-an-external-http-service}

1. `ServiceEntry` を作成し、外部 HTTP サービスへのアクセスを許可します：

   {{< warning >}}
   下記のサービスエントリでは `DNS` 解決を使っています。`NONE` にするとセキュリティリスクがあります。
   悪意のあるクライアントが本来とは異なる IP に接続しつつ `HOST` ヘッダを `httpbin.org` に偽装する可能性があります。
   Istio Sidecar プロキシは HOST ヘッダを信頼し、誤って通信を許可し、他のホストの IP アドレスに転送してしまうことがあります。

   そのホストが悪意のあるサイトや、メッシュのセキュリティポリシーで禁止されているサイトである可能性もあります。

   `DNS` 解決を使うことで、Sidecar プロキシは元の宛先 IP を無視し、`httpbin.org` の DNS 解決結果にトラフィックを誘導します。
   {{< /warning >}}

   {{< text bash >}}
   $ kubectl apply -f - <<EOF
   apiVersion: networking.istio.io/v1
   kind: ServiceEntry
   metadata:
   name: httpbin-ext
   spec:
   hosts:

   - httpbin.org
     ports:
   - number: 80
     name: http
     protocol: HTTP
     resolution: DNS
     location: MESH_EXTERNAL
     EOF
     {{< /text >}}

1. `SOURCE_POD` から外部 HTTP サービスにリクエストします：

   {{< text bash >}}
   $ kubectl exec "$SOURCE_POD" -c curl -- curl -sS http://httpbin.org/headers
   {
   "headers": {
   "Accept": "_/_",
   "Host": "httpbin.org",
   ...
   "X-Envoy-Decorator-Operation": "httpbin.org:80/\*",
   ...
   }
   }
   {{< /text >}}

   Istio Sidecar プロキシによって追加された `X-Envoy-Decorator-Operation` ヘッダに注目してください。

1. `SOURCE_POD` の Sidecar プロキシのログを確認します：

   {{< text bash >}}
   $ kubectl logs "$SOURCE_POD" -c istio-proxy | tail
   [2019-01-24T12:17:11.640Z] "GET /headers HTTP/1.1" 200 - 0 599 214 214 "-" "curl/7.60.0" "17fde8f7-fa62-9b39-8999-302324e6def2" "httpbin.org" "35.173.6.94:80" outbound|80||httpbin.org - 35.173.6.94:80 172.30.109.82:55314 -
   {{< /text >}}

   HTTP リクエストに関連する `httpbin.org/headers` を確認してください。

### 外部 HTTPS サービスへのアクセス {#access-an-external-https-service}

1. `ServiceEntry` を作成し、外部サービスへのアクセスを許可します。

   {{< text bash >}}
   $ kubectl apply -f - <<EOF
   apiVersion: networking.istio.io/v1
   kind: ServiceEntry
   metadata:
   name: google
   spec:
   hosts:

   - www.google.com
     ports:
   - number: 443
     name: https
     protocol: HTTPS
     resolution: DNS
     location: MESH_EXTERNAL
     EOF
     {{< /text >}}

1. `SOURCE_POD` から外部 HTTPS サービスにリクエストします：

   {{< text bash >}}
   $ kubectl exec "$SOURCE_POD" -c curl -- curl -sSI https://www.google.com | grep "HTTP/"
   HTTP/2 200
   {{< /text >}}

1. `SOURCE_POD` の Sidecar プロキシのログを確認します：

   {{< text bash >}}
   $ kubectl logs "$SOURCE_POD" -c istio-proxy | tail
   [2019-01-24T12:48:54.977Z] "- - -" 0 - 601 17766 1289 - "-" "-" "-" "-" "172.217.161.36:443" outbound|443||www.google.com 172.30.109.82:59480 172.217.161.36:443 172.30.109.82:59478 www.google.com
   {{< /text >}}

   `www.google.com` への HTTPS リクエストに関連するエントリに注目してください。

### 外部サービスへのトラフィック管理 {#manage-traffic-to-external-services}

クラスタ間リクエストと同様に、`ServiceEntry` で設定した外部サービスにもルーティングルールを設定できます。
ここでは、`httpbin.org` サービスへのアクセスにタイムアウトルールを設定します。

{{< boilerplate gateway-api-support >}}

1. テストソース Pod から外部サービス `httpbin.org` の `/delay` エンドポイントに **curl** リクエストを送信します：

   {{< text bash >}}
   $ kubectl exec "$SOURCE_POD" -c curl -- time curl -o /dev/null -sS -w "%{http_code}\n" http://httpbin.org/delay/5
   200
   real 0m5.024s
   user 0m0.003s
   sys 0m0.003s
   {{< /text >}}

   このリクエストは約 5 秒で 200 (OK) を返します。

2. `kubectl` で外部サービス `httpbin.org` へのタイムアウトを 3 秒に設定します。

{{< tabset category-name="config-api" >}}

{{< tab name="Istio API" category-value="istio-apis" >}}

{{< text bash >}}
$ kubectl apply -f - <<EOF
apiVersion: networking.istio.io/v1
kind: VirtualService
metadata:
name: httpbin-ext
spec:
hosts:

- httpbin.org
  http:
- timeout: 3s
  route: - destination:
  host: httpbin.org
  weight: 100
  EOF
  {{< /text >}}

{{< /tab >}}

{{< tab name="Gateway API" category-value="gateway-api" >}}

{{< text bash >}}
$ kubectl apply -f - <<EOF
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
name: httpbin-ext
spec:
parentRefs:

- kind: ServiceEntry
  group: networking.istio.io
  name: httpbin-ext
  hostnames:
- httpbin.org
  rules:
- timeouts:
  request: 3s
  backendRefs: - kind: Hostname
  group: networking.istio.io
  name: httpbin.org
  port: 80
  EOF
  {{< /text >}}

{{< /tab >}}

{{< /tabset >}}

3. 数秒後、再度 **curl** リクエストを送信します：

   {{< text bash >}}
   $ kubectl exec "$SOURCE_POD" -c curl -- time curl -o /dev/null -sS -w "%{http_code}\n" http://httpbin.org/delay/5
   504
   real 0m3.149s
   user 0m0.004s
   sys 0m0.004s
   {{< /text >}}

   今回は 3 秒後に 504 (Gateway Timeout) となります。Istio が 3 秒で `httpbin.org` への応答を切断しました。

### 外部サービスへの制御付きアクセスのクリーンアップ {#cleanup-the-controlled-access-to-external-services}

{{< tabset category-name="config-api" >}}

{{< tab name="Istio API" category-value="istio-apis" >}}

{{< text bash >}}
$ kubectl delete serviceentry httpbin-ext google
$ kubectl delete virtualservice httpbin-ext --ignore-not-found=true
{{< /text >}}

{{< /tab >}}

{{< tab name="Gateway API" category-value="gateway-api" >}}

{{< text bash >}}
$ kubectl delete serviceentry httpbin-ext
$ kubectl delete httproute httpbin-ext --ignore-not-found=true
{{< /text >}}

{{< /tab >}}

{{< /tabset >}}

## 外部サービスへの直接アクセス {#direct-access-to-external-services}

特定の IP 範囲については、Istio を完全にバイパスするように Envoy Sidecar を設定できます。これにより、[トラフィックのインターセプト](/ja/docs/concepts/traffic-management/) を防ぎます。バイパス設定は `global.proxy.includeIPRanges` または `global.proxy.excludeIPRanges` [設定パラメータ](https://archive.istio.io/v1.4/docs/reference/config/installation-options/) で行い、`istio-sidecar-injector` 設定を `kubectl apply` で更新します。また、Pod の[アノテーション](/ja/docs/reference/config/annotations/)（例：`traffic.sidecar.istio.io / includeOutboundIPRanges`）でも設定できます。`istio-sidecar-injector` 設定の更新は新規デプロイ Pod に影響します。

{{< warning >}}
[Envoy で外部サービスへパススルー](#envoy-passthrough-to-external-services) とは異なり、`ALLOW_ANY` トラフィックポリシーで Sidecar プロキシが未知のサービスへの呼び出しをパススルーする場合と違い、この方法は Sidecar を完全にバイパスし、指定した IP への Istio 機能を無効化します。`ALLOW_ANY` のように特定の宛先だけ Service Entry を追加することはできません。
パフォーマンスやその他の理由で Sidecar 経由の外部アクセスができない場合のみ、この方法を推奨します。
{{< /warning >}}

すべての外部 IP へのリダイレクトを Sidecar プロキシから除外するには、`global.proxy.includeIPRanges` 設定オプションをクラスタ内サービスで使われている IP 範囲に設定します。IP 範囲はクラスタのプラットフォームによって異なります。

### プラットフォームごとの内部 IP 範囲の確認 {#determine-the-internal-IP-ranges-for-your-platform}

クラスタプロバイダに応じて `global.proxy.includeIPRanges` パラメータを設定します。

#### IBM Cloud Private

1. `IBM Cloud Private` の設定ファイル `cluster/config.yaml` から `service_cluster_ip_range` を取得します：

   {{< text bash >}}
    $ grep service_cluster_ip_range cluster/config.yaml
   {{< /text >}}

   出力例：

   {{< text plain >}}
    service_cluster_ip_range: 10.0.0.1/24
   {{< /text >}}

1. `--set global.proxy.includeIPRanges="10.0.0.1/24"` を使用します。

#### IBM Cloud Kubernetes Service

クラスタで使われている CIDR を確認するには、`ibmcloud ks cluster get -c <CLUSTER-NAME>` で `Service Subnet` を参照します：

{{< text bash >}}
$ ibmcloud ks cluster get -c my-cluster | grep "Service Subnet"
Service Subnet: 172.21.0.0/16
{{< /text >}}

`--set values.global.proxy.includeIPRanges="172.21.0.0/16"` を使用します。

{{< warning >}}
古いクラスタでは正しく動作しない場合があるため、
`--set values.global.proxy.includeIPRanges="172.30.0.0/16,172.21.0.0/16,10.10.10.0/24"`
や `kubectl get svc -o wide -A` で CIDR を確認してください。
{{< /warning >}}

#### Google Kubernetes Engine (GKE)

範囲は固定ではないため、`gcloud container clusters describe` で確認します。例：

{{< text bash >}}
$ gcloud container clusters describe XXXXXXX --zone=XXXXXX | grep -e clusterIpv4Cidr -e servicesIpv4Cidr
clusterIpv4Cidr: 10.4.0.0/14
servicesIpv4Cidr: 10.7.240.0/20
{{< /text >}}

`--set global.proxy.includeIPRanges="10.4.0.0/14\,10.7.240.0/20"` を使用します。

#### Azure Kubernetes Service (AKS)

##### Kubenet

クラスタで使われている Service CIDR と Pod CIDR を確認するには、`az aks show` で `serviceCidr` を参照します：

{{< text bash >}}
$ az aks show --resource-group "${RESOURCE_GROUP}" --name "${CLUSTER}" | grep Cidr
"podCidr": "10.244.0.0/16",
"podCidrs": [
"serviceCidr": "10.0.0.0/16",
"serviceCidrs": [
{{< /text >}}

`--set values.global.proxy.includeIPRanges="10.244.0.0/16\,10.0.0.0/16"` を使用します。

##### Azure CNI

Azure CNI でオーバーレイネットワークを使わない場合は以下の手順で確認します。
オーバーレイネットワークの場合は [Kubenet の説明](#kubenet) を参照してください。
詳細は [Azure CNI Overlay ドキュメント](https://learn.microsoft.com/en-us/azure/aks/azure-cni-overlay) を参照してください。

Service CIDR を確認するには `az aks show` で `serviceCidr` を参照します：

{{< text bash >}}
$ az aks show --resource-group "${RESOURCE_GROUP}" --name "${CLUSTER}" | grep serviceCidr
"serviceCidr": "10.0.0.0/16",
"serviceCidrs": [
{{< /text >}}

Pod CIDR を確認するには `az` CLI で `vnet` を調べます：

{{< text bash >}}
$ az aks show --resource-group "${RESOURCE_GROUP}" --name "${CLUSTER}" | grep nodeResourceGroup
"nodeResourceGroup": "MC_user-rg_user-cluster_region",
"nodeResourceGroupProfile": null,
$ az network vnet list -g MC_user-rg_user-cluster_region | grep name
"name": "aks-vnet-74242220",
"name": "aks-subnet",
$ az network vnet show -g MC_user-rg_user-cluster_region -n aks-vnet-74242220 | grep addressPrefix
"addressPrefixes": [
"addressPrefix": "10.224.0.0/16",
{{< /text >}}

`--set values.global.proxy.includeIPRanges="10.244.0.0/16\,10.0.0.0/16"` を使用します。

#### Minikube, Docker For Desktop, Bare Metal

デフォルト値は `10.96.0.0/12` ですが、固定ではありません。次のコマンドで実際の値を確認してください：

{{< text bash >}}
$ kubectl describe pod kube-apiserver -n kube-system | grep 'service-cluster-ip-range'
--service-cluster-ip-range=10.96.0.0/12
{{< /text >}}

`--set global.proxy.includeIPRanges="10.96.0.0/12"` を使用します。

### プロキシバイパスの設定 {#configuring-the-proxy-bypass}

{{< warning >}}
このガイドで以前にデプロイした Service Entry や Virtual Service を削除してください。
{{< /warning >}}

プラットフォームの IP 範囲で `istio-sidecar-injector` の設定を更新します。例えば IP 範囲が 10.0.0.1/24 の場合：

{{< text syntax=bash snip_id=none >}}
$ istioctl install <flags-you-used-to-install-Istio> --set values.global.proxy.includeIPRanges="10.0.0.1/24"
{{< /text >}}

[Istio のインストール](/ja/docs/setup/install/istioctl) コマンドに `--set values.global.proxy.includeIPRanges="10.0.0.1/24"` を追加してください。

### 外部サービスへのアクセス {#access-the-external-services}

バイパス設定は新規デプロイにのみ影響するため、[始める前に](#before-you-begin)の手順に従って `curl` プログラムを再デプロイしてください。

`istio-sidecar-injector` の configmap を更新し、`curl` プログラムを再デプロイした後、Istio Sidecar はクラスタ内の内部リクエストのみをインターセプト・管理します。外部リクエストは Sidecar をバイパスし、直接目的地に到達します。例：

{{< text bash >}}
$ kubectl exec "$SOURCE_POD" -c curl -- curl -sS http://httpbin.org/headers
{
"headers": {
"Accept": "_/_",
"Host": "httpbin.org",
...
}
}
{{< /text >}}

HTTP/HTTPS で外部サービスにアクセスした場合と異なり、Istio Sidecar に関連するヘッダは付与されず、外部サービスへのリクエストは Sidecar や Mixer のログにも記録されません。Sidecar をバイパスすると、外部サービスへのアクセス監視ができなくなります。

### 外部サービスへの直接アクセスのクリーンアップ {#cleanup-the-direct-access-to-external-services}

設定を更新し、さまざまな IP への Sidecar バイパスを停止します：

{{< text syntax=bash snip_id=none >}}
$ istioctl install <flags-you-used-to-install-Istio>
{{< /text >}}

## 仕組みの理解 {#understanding-what-happened}

このタスクでは、Istio メッシュから外部サービスを呼び出す 3 つの方法を学びました：

1. Envoy を設定して任意の外部サービスへのアクセスを許可する方法。

1. Service Entry を使ってアクセス可能な外部サービスをメッシュに登録する方法（推奨）。

1. Istio Sidecar で外部 IP をリマップ対象から除外する方法。

1 つ目の方法は、Istio Sidecar プロキシ経由でメッシュ内の未知サービスへのトラフィックも誘導します。この方法では外部サービスへのアクセス監視や Istio のトラフィック制御機能は利用できません。特定サービスだけ 2 つ目の方法に切り替えたい場合は、その外部サービスの Service Entry を作成するだけで済みます。

2 つ目の方法では、Istio サービスメッシュのすべての機能を使ってクラスタ内外のサービスを呼び出せます。このタスクでは、外部サービスへのアクセス監視やタイムアウトルールの設定方法を学びました。

3 つ目の方法は、Istio Sidecar プロキシをバイパスして任意の外部サービスに直接アクセスします。ただし、この方法にはクラスタプロバイダ固有の知識や設定が必要です。1 つ目の方法と同様、外部サービスへのアクセス監視や Istio 機能の適用はできません。

## セキュリティノート {#security-note}

{{< warning >}}
このタスクの設定例では**安全な外部トラフィック制御は有効化されていません**。
悪意のあるプログラムは Istio Sidecar プロキシをバイパスして、Istio の制御なしに任意の外部サービスにアクセスできます。
{{< /warning >}}

より安全に外部トラフィック制御を行うには、[Egress ゲートウェイ経由で外部トラフィックを誘導](/ja/docs/tasks/traffic-management/egress/egress-gateway/)し、[追加のセキュリティ考慮事項](/ja/docs/tasks/traffic-management/egress/egress-gateway/#additional-security-considerations)も参照してください。

## クリーンアップ {#cleanup}

[curl]({{< github_tree >}}/samples/curl) サービスを削除します：

{{< text bash >}}
$ kubectl delete -f @samples/curl/curl.yaml@
{{< /text >}}
