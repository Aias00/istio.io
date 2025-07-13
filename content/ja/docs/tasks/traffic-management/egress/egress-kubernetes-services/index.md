---
title: Kubernetes 外部サービスへの出口トラフィック
description: Istio で Kubernetes 外部サービスを構成する方法を紹介します。
keywords: [traffic-management, egress]
weight: 60
owner: istio/wg-networking-maintainers
test: yes
---

Kubernetes の [ExternalName](https://kubernetes.io/ja/docs/concepts/services-networking/service/#externalname) サービスや [Endpoints](https://kubernetes.io/ja/docs/concepts/services-networking/service/#services-without-selectors) を持つ Kubernetes サービスを使うことで、外部サービスへのローカル DNS エイリアスを作成できます。この DNS エイリアスはローカルサービスの DNS エントリと同じ形式、すなわち `<service name>.<namespace name>.svc.cluster.local` となります。DNS エイリアスはワークロードに「ロケーションの透過性」を提供します：ワークロードはローカルサービスと外部サービスを同じ方法で呼び出せます。将来的に外部サービスをクラスタ内にデプロイする場合でも、Kubernetes サービスをローカルバージョンに更新するだけで済みます。ワークロード側の変更は不要です。

このページでは、これらの外部サービスへのアクセスのための Kubernetes の仕組みが Istio でも有効であることを示します。TLS モードを使うだけで、Istio の[双方向 TLS](/ja/docs/concepts/security/#mutual-TLS-authentication)は不要です。外部サービスは Istio サービスメッシュの一部ではないため、Istio の双方向 TLS を実施できません。TLS モードの設定は、外部サービスの TLS モード要件と、ワークロードが外部サービスにアクセスする方法に従ってください。ワークロードが HTTP リクエストを発行し、外部サービスが TLS を必要とする場合は、Istio で TLS オリジネーションを行えます。ワークロードがすでに TLS でトラフィックを暗号化している場合は、Istio の双方向 TLS を無効にできます。

{{< warning >}}
このページは既存の Kubernetes 設定との統合方法を説明しています。新規デプロイの場合は、[外部サービスへのアクセス](/ja/docs/tasks/traffic-management/egress/egress-control/) の手順に従うことを推奨します。
{{< /warning >}}

このページの例は HTTP プロトコルを使っていますが、Kubernetes サービスを使った出口トラフィックの誘導は他のプロトコルでも利用できます。

{{< boilerplate before-you-begin-egress >}}

- Istio 管理外のソース Pod 用に名前空間を作成します：

  {{< text bash >}}
    $ kubectl create namespace without-istio
  {{< /text >}}

- `without-istio` 名前空間で [curl]({{< github_tree >}}/samples/curl) サンプルを起動します。

  {{< text bash >}}
    $ kubectl apply -f @samples/curl/curl.yaml@ -n without-istio
  {{< /text >}}

- リクエスト送信のため、環境変数 `SOURCE_POD_WITHOUT_ISTIO` にソース Pod の名前を保存します：

  {{< text bash >}}
    $ export SOURCE_POD_WITHOUT_ISTIO="$(kubectl get pod -n without-istio -l app=curl -o jsonpath={.items..metadata.name})"
  {{< /text >}}

- Istio Sidecar が注入されていないこと（Pod にコンテナが 1 つだけであること）を確認します：

  {{< text bash >}}
    $ kubectl get pod "$SOURCE_POD_WITHOUT_ISTIO" -n without-istio
    NAME READY STATUS RESTARTS AGE
    curl-66c8d79ff5-8tqrl 1/1 Running 0 32s
  {{< /text >}}

## Kubernetes ExternalName サービスで外部サービスにアクセスする {#ks-external-name-service-to-access-an-external-service}

1. デフォルト名前空間で `httpbin.org` 用の Kubernetes [ExternalName](https://kubernetes.io/ja/docs/concepts/services-networking/service/#externalname) サービスを作成します：

   {{< text bash >}}
    $ kubectl apply -f - <<EOF
    kind: Service
    apiVersion: v1
    metadata:
    name: my-httpbin
    spec:
    type: ExternalName
    externalName: httpbin.org
    ports:

    - name: http
      protocol: TCP
      port: 80
      EOF
     {{< /text >}}

1. サービスを確認します。クラスタ IP がないことに注意してください。

   {{< text bash >}}
    $ kubectl get svc my-httpbin
    NAME TYPE CLUSTER-IP EXTERNAL-IP PORT(S) AGE
    my-httpbin ExternalName <none> httpbin.org 80/TCP 4s
   {{< /text >}}

1. Istio Sidecar のないソース Pod から Kubernetes サービスのホスト名経由で `httpbin.org` にアクセスします。下記の **curl** コマンドは [Kubernetes サービスの DNS 形式](https://v1-13.docs.kubernetes.io/docs/concepts/services-networking/dns-pod-service/#a-records)（`<service name>.<namespace>.svc.cluster.local`）を使っています。

   {{< text bash >}}
    $ kubectl exec "$SOURCE_POD_WITHOUT_ISTIO" -n without-istio -c curl -- curl -sS my-httpbin.default.svc.cluster.local/headers
    {
    "headers": {
    "Accept": "_/_",
    "Host": "my-httpbin.default.svc.cluster.local",
    "User-Agent": "curl/7.55.0"
    }
    }
   {{< /text >}}

1. この例では、暗号化されていない HTTP リクエストが `httpbin.org` に送信されます。サンプルのため TLS モードを無効化し、外部サービスへの平文トラフィックを許可しています。実際には Istio で [Egress TLS オリジネーション](/ja/docs/tasks/traffic-management/egress/egress-tls-origination) を推奨します。

   {{< text bash >}}
    $ kubectl apply -f - <<EOF
    apiVersion: networking.istio.io/v1
    kind: DestinationRule
    metadata:
    name: my-httpbin
    spec:
    host: my-httpbin.default.svc.cluster.local
    trafficPolicy:
    tls:
    mode: DISABLE
    EOF
   {{< /text >}}

1. Istio Sidecar 付きのソース Pod から Kubernetes サービスのホスト名経由で `httpbin.org` にアクセスします。Istio Sidecar によって追加されるヘッダ（例：`X-Istio-Attributes` や `X-Envoy-Peer-Metadata`）に注目してください。また、`Host` ヘッダはサービスのホスト名になっています。

   {{< text bash >}}
    $ kubectl exec "$SOURCE_POD" -c curl -- curl -sS my-httpbin.default.svc.cluster.local/headers
    {
    "headers": {
    "Accept": "_/_",
    "Content-Length": "0",
    "Host": "my-httpbin.default.svc.cluster.local",
    "User-Agent": "curl/7.64.0",
    "X-B3-Sampled": "0",
    "X-B3-Spanid": "5795fab599dca0b8",
    "X-B3-Traceid": "5079ad3a4af418915795fab599dca0b8",
    "X-Envoy-Peer-Metadata": "...",
    "X-Envoy-Peer-Metadata-Id": "sidecar~10.28.1.74~curl-6bdb595bcb-drr45.default~default.svc.cluster.local"
    }
    }
   {{< /text >}}

### Kubernetes ExternalName サービスのクリーンアップ {#cleanup-of-ks-external-name-service}

{{< text bash >}}
$ kubectl delete destinationrule my-httpbin
$ kubectl delete service my-httpbin
{{< /text >}}

## Endpoints を持つ Kubernetes サービスで外部サービスにアクセスする {#use-a-ks-service-with-endpoints-to-access-an-external-service}

1. Wikipedia 用に selector なしの Kubernetes サービスを作成します：

   {{< text bash >}}
    $ kubectl apply -f - <<EOF
    kind: Service
    apiVersion: v1
    metadata:
    name: my-wikipedia
    spec:
    ports:
 
    - protocol: TCP
      port: 443
      name: tls
      EOF
     {{< /text >}}

1. サービス用の endpoints を作成します。[Wikipedia の IP 範囲リスト](https://www.mediawiki.org/wiki/Wikipedia_Zero/IP_Addresses) からいくつかの IP を選びます。

   {{< text bash >}}
    $ kubectl apply -f - <<EOF
    kind: Endpoints
    apiVersion: v1
    metadata:
    name: my-wikipedia
    subsets:
 
    - addresses: - ip: 198.35.26.96 - ip: 208.80.153.224
      ports: - port: 443
      name: tls
      EOF
     {{< /text >}}

1. サービスを確認します。クラスタ IP が割り当てられており、それを使って `wikipedia.org` にアクセスできます。

   {{< text bash >}}
    $ kubectl get svc my-wikipedia
    NAME TYPE CLUSTER-IP EXTERNAL-IP PORT(S) AGE
    my-wikipedia ClusterIP 172.21.156.230 <none> 443/TCP 21h
   {{< /text >}}

1. Istio Sidecar のないソース Pod から Kubernetes サービスのクラスタ IP 経由で `wikipedia.org` へ HTTPS リクエストを送信します。`curl` の `--resolve` オプションを使い、クラスタ IP 経由で `wikipedia.org` にアクセスします：

   {{< text bash >}}
    $ kubectl exec "$SOURCE_POD_WITHOUT_ISTIO" -n without-istio -c curl -- curl -sS --resolve en.wikipedia.org:443:"$(kubectl get service my-wikipedia -o jsonpath='{.spec.clusterIP}')" https://en.wikipedia.org/wiki/Main_Page | grep -o "<title>.\*</title>"
   <title>Wikipedia, the free encyclopedia</title>
   {{< /text >}}

1. この場合、ワークロードは HTTPS リクエスト（TLS 接続）を `wikipedia.org` に送信します。トラフィックはワークロード側で暗号化されているため、Istio の双方向 TLS を無効にしても安全です：

   {{< text bash >}}
    $ kubectl apply -f - <<EOF
    apiVersion: networking.istio.io/v1
    kind: DestinationRule
    metadata:
    name: my-wikipedia
    spec:
    host: my-wikipedia.default.svc.cluster.local
    trafficPolicy:
    tls:
    mode: DISABLE
    EOF
   {{< /text >}}

1. Istio Sidecar 付きのソース Pod から Kubernetes サービスのクラスタ IP 経由で `wikipedia.org` にアクセスします：

   {{< text bash >}}
    $ kubectl exec "$SOURCE_POD" -c curl -- curl -sS --resolve en.wikipedia.org:443:"$(kubectl get service my-wikipedia -o jsonpath='{.spec.clusterIP}')" https://en.wikipedia.org/wiki/Main_Page | grep -o "<title>.\*</title>"
   <title>Wikipedia, the free encyclopedia</title>
   {{< /text >}}

1. アクセスが本当にクラスタ IP 経由で行われているか確認します。`curl -v` の出力で `Connected to en.wikipedia.org (172.21.156.230)` という行があり、サービス出力で表示されたクラスタ IP が使われていることが分かります。

   {{< text bash >}}
    $ kubectl exec "$SOURCE_POD" -c curl -- curl -sS -v --resolve en.wikipedia.org:443:"$(kubectl get service my-wikipedia -o jsonpath='{.spec.clusterIP}')" https://en.wikipedia.org/wiki/Main_Page -o /dev/null

    - Added en.wikipedia.org:443:172.21.156.230 to DNS cache
    - Hostname en.wikipedia.org was found in DNS cache
    - Trying 172.21.156.230...
    - TCP_NODELAY set
    - Connected to en.wikipedia.org (172.21.156.230) port 443 (#0)
      ...
     {{< /text >}}

### Endpoints を持つ Kubernetes サービスのクリーンアップ {#cleanup-of-ks-service-with-endpoints}

{{< text bash >}}
$ kubectl delete destinationrule my-wikipedia
$ kubectl delete endpoints my-wikipedia
$ kubectl delete service my-wikipedia
{{< /text >}}

## クリーンアップ {#cleanup}

1. [curl]({{< github_tree >}}/samples/curl) サービスを停止します：

   {{< text bash >}}
    $ kubectl delete -f @samples/curl/curl.yaml@
   {{< /text >}}

1. `without-istio` 名前空間の [curl]({{< github_tree >}}/samples/curl) サービスを停止します：

   {{< text bash >}}
    $ kubectl delete -f @samples/curl/curl.yaml@ -n without-istio
   {{< /text >}}

1. `without-istio` 名前空間を削除します：

   {{< text bash >}}
    $ kubectl delete namespace without-istio
   {{< /text >}}

1. 環境変数を解除します：

   {{< text bash >}}
    $ unset SOURCE_POD SOURCE_POD_WITHOUT_ISTIO
   {{< /text >}}
