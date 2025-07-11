---
title: Kubernetes Ingress
description: Kubernetes Ingress オブジェクトを構成して、サービスメッシュ外部からメッシュ内サービスへアクセスする方法を示します。
weight: 40
keywords: [traffic-management, ingress]
owner: istio/wg-networking-maintainers
test: yes
---

このタスクでは、[Kubernetes Ingress](https://kubernetes.io/ja/docs/concepts/services-networking/ingress/) を使って、Istio でサービスメッシュクラスタ内のサービスを外部に公開するためのエントリーゲートウェイを構成する方法を説明します。

{{< tip >}}
Istio の豊富なトラフィック管理やセキュリティ機能を活用するには、Ingress よりも [Gateway](/ja/docs/tasks/traffic-management/ingress/ingress-control/) の利用を推奨します。
{{< /tip >}}

## 準備 {#before-you-begin}

[エントリーゲートウェイのタスク](/ja/docs/tasks/traffic-management/ingress/ingress-control/)の [準備](/ja/docs/tasks/traffic-management/ingress/ingress-control/#before-you-begin) および [Ingress IP とポートの確認](/ja/docs/tasks/traffic-management/ingress/ingress-control/#determining-the-ingress-ip-and-ports) の手順に従ってください。

## Ingress リソースでエントリーゲートウェイを構成する {#configuring-ingress-using-an-ingress-resource}

[Kubernetes Ingress](https://kubernetes.io/ja/docs/concepts/services-networking/ingress/) は、クラスタ外部からクラスタ内サービスへの HTTP および HTTPS ルーティングを公開します。

ここでは、ポート 80 で HTTP トラフィック用に `Ingress` を構成する方法を見てみましょう。

1.  `Ingress` リソースと `IngressClass` を作成します：

    {{< text bash >}}
    $ kubectl apply -f - <<EOF
    apiVersion: networking.k8s.io/v1
    kind: IngressClass
    metadata:
    name: istio
    spec:
    controller: istio.io/ingress-controller

    ***

    apiVersion: networking.k8s.io/v1
    kind: Ingress
    metadata:
    name: ingress
    spec:
    ingressClassName: istio
    rules:

    - host: httpbin.example.com
      http:
      paths: - path: /status
      pathType: Prefix
      backend:
      service:
      name: httpbin
      port:
      number: 8000
      EOF
      {{< /text >}}

    `IngressClass` リソースは Kubernetes に Istio ゲートウェイコントローラーを示し、
    `ingressClassName: istio` の値は Kubernetes に Istio ゲートウェイコントローラーがこの `Ingress` を処理すべきであることを指示します。

    古いバージョンの Ingress API では `kubernetes.io/ingress.class` アノテーションが使われていましたが、
    これは依然として有効ですが [Kubernetes で非推奨](https://kubernetes.io/ja/docs/concepts/services-networking/ingress/#deprecated-annotation) となっています。

1.  **curl** を使って **httpbin** サービスにアクセスします：

    {{< text bash >}}
    $ curl -s -I -HHost:httpbin.example.com "http://$INGRESS_HOST:$INGRESS_PORT/status/200"
    ...
    HTTP/1.1 200 OK
    ...
    server: istio-envoy
    ...
    {{< /text >}}

    `-H` フラグで **Host** HTTP ヘッダーを "httpbin.example.com" に設定する必要があります。
    これは `Ingress` で "httpbin.example.com" へのリクエストを処理するよう構成されているためですが、テスト環境ではこのホストに DNS バインドがないためです。

1.  明示的に公開されていない他の URL にアクセスすると、HTTP 404 エラーが返されます：

    {{< text bash >}}
    $ curl -s -I -HHost:httpbin.example.com "http://$INGRESS_HOST:$INGRESS_PORT/headers"
    HTTP/1.1 404 Not Found
    ...
    {{< /text >}}

## 次のステップ {#next-steps}

### TLS {#TLS}

`Ingress` は [TLS 設定の指定](https://kubernetes.io/ja/docs/concepts/services-networking/ingress/#tls) をサポートしています。
Istio もこの機能をサポートしていますが、参照される `Secret` は `istio-ingressgateway` がデプロイされている名前空間（通常は `istio-system`）に存在する必要があります。
[cert-manager](/ja/docs/ops/integrations/certmanager/) を使ってこれらの証明書を生成できます。

### パスタイプの指定 {#specifying-path-type}

Istio のデフォルトのパスタイプは厳密一致ですが、パスが `/*` または `.*` で終わる場合はプレフィックス一致となります。他の正規表現はサポートされていません。

Kubernetes 1.18 で新しいフィールド `pathType` が追加されました。これにより、パスを明示的に `Exact` または `Prefix` として宣言できます。

## クリーンアップ {#cleanup}

`IngressClass` と `Ingress` の構成を削除し、[httpbin]({{< github_tree >}}/samples/httpbin) サービスを停止します：

{{< text bash >}}
$ kubectl delete ingress ingress
$ kubectl delete ingressclass istio
$ kubectl delete --ignore-not-found=true -f @samples/httpbin/httpbin.yaml@
{{< /text >}}
