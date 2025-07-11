---
title: 外部 HTTPS プロキシの利用
description: Istio でアプリケーションが外部 HTTPS プロキシを利用できるように構成する方法を説明します。
weight: 60
keywords: [traffic-management, egress]
aliases:
  - /zh/docs/examples/advanced-gateways/http-proxy/
owner: istio/wg-networking-maintainers
test: yes
---

[egress ゲートウェイの構成](/ja/docs/tasks/traffic-management/egress/egress-gateway/) の例では、Istio の egress ゲートウェイコンポーネントを使ってメッシュ外のサービスへトラフィックを誘導する方法を紹介しました。しかし、場合によっては外部の従来型（非 Istio）HTTPS プロキシを経由して外部サービスへアクセスする必要があります。たとえば、企業内に既存のプロキシがあり、すべてのアプリケーションがそのプロキシ経由でトラフィックを送る必要がある場合などです。

この例では、外部 HTTPS プロキシへのアクセスを有効にする方法を説明します。アプリケーションは HTTP の [CONNECT](https://tools.ietf.org/html/rfc7231#section-4.3.6) メソッドを使って HTTPS プロキシと接続を確立するため、外部 HTTP/HTTPS サービスへのトラフィック構成とは異なります。

{{< boilerplate before-you-begin-egress >}}

- [Envoy のアクセスログを有効化](/ja/docs/tasks/observability/logs/access-log/#enable-envoy-s-access-logging)

## HTTPS プロキシのデプロイ {#deploy-an-https-proxy}

この例では従来型プロキシを模倣するため、クラスタ内に HTTPS プロキシをデプロイします。また、より現実的な「クラスタ外」プロキシを模擬するため、Kubernetes サービスのドメイン名ではなくプロキシ Pod の IP アドレスでプロキシを指定します。ここでは [squid](http://www.squid-cache.org) を使いますが、HTTP CONNECT に対応した任意の HTTPS プロキシが利用可能です。

1. HTTPS プロキシ用の名前空間を作成します。Sidecar 注入用のラベルを付与しなければ、この名前空間では Sidecar 注入が無効となり、Istio でこの名前空間のトラフィックは制御されません。これによりクラスタ外のプロキシを模擬できます。

   {{< text bash >}}
   $ kubectl create namespace external
   {{< /text >}}

1. Squid プロキシの設定ファイルを作成します。

   {{< text bash >}}
   $ cat <<EOF > ./proxy.conf
   http_port 3128

   acl SSL_ports port 443
   acl CONNECT method CONNECT

   http_access deny CONNECT !SSL_ports
   http_access allow localhost manager
   http_access deny manager
   http_access allow all

   coredump_dir /var/spool/squid
   EOF
   {{< /text >}}

1. Kubernetes [ConfigMap](https://kubernetes.io/ja/docs/tasks/configure-pod-container/configure-pod-configmap/) を作成し、プロキシの設定を保存します：

   {{< text bash >}}
   $ kubectl create configmap proxy-configmap -n external --from-file=squid.conf=./proxy.conf
   {{< /text >}}

1. Squid コンテナをデプロイします：

   {{< text bash >}}
   $ kubectl apply -f - <<EOF
   apiVersion: apps/v1
   kind: Deployment
   metadata:
   name: squid
   namespace: external
   spec:
   replicas: 1
   selector:
   matchLabels:
   app: squid
   template:
   metadata:
   labels:
   app: squid
   spec:
   volumes: - name: proxy-config
   configMap:
   name: proxy-configmap
   containers: - name: squid
   image: sameersbn/squid:3.5.27
   imagePullPolicy: IfNotPresent
   volumeMounts: - name: proxy-config
   mountPath: /etc/squid
   readOnly: true
   EOF
   {{< /text >}}

1. `external` 名前空間に [curl]({{< github_tree >}}/samples/curl) サンプルをデプロイし、プロキシ経由の通信をテストします（Istio のトラフィック制御は行われません）。

   {{< text bash >}}
   $ kubectl apply -n external -f @samples/curl/curl.yaml@
   {{< /text >}}

1. プロキシ Pod の IP アドレスを取得し、`PROXY_IP` 環境変数に格納します：

   {{< text bash >}}
   $ export PROXY_IP=$(kubectl get pod -n external -l app=squid -o jsonpath={.items..podIP})
   {{< /text >}}

1. プロキシのポート番号（この例では 3128）を `PROXY_PORT` 環境変数に格納します。

   {{< text bash >}}
   $ export PROXY_PORT=3128
   {{< /text >}}

1. `external` 名前空間の curl Pod からプロキシ経由で外部サービスにリクエストを送信します：

   {{< text bash >}}
   $ kubectl exec -it $(kubectl get pod -n external -l app=curl -o jsonpath={.items..metadata.name}) -n external -- sh -c "HTTPS_PROXY=$PROXY_IP:$PROXY_PORT curl https://en.wikipedia.org/wiki/Main_Page" | grep -o "<title>.\*</title>"
   <title>Wikipedia, the free encyclopedia</title>
   {{< /text >}}

1. プロキシのアクセスログを確認します：

   {{< text bash >}}
   $ kubectl exec -it $(kubectl get pod -n external -l app=squid -o jsonpath={.items..metadata.name}) -n external -- tail -f /var/log/squid/access.log
   1544160065.248 228 172.30.109.89 TCP_TUNNEL/200 87633 CONNECT en.wikipedia.org:443 - HIER_DIRECT/91.198.174.192 -
   {{< /text >}}

これで Istio を使わずに以下のことができました：

- HTTPS プロキシをデプロイした
- curl でプロキシ経由で wikipedia.org へアクセスした

次に、Istio 対応 Pod のトラフィックを HTTPS プロキシ経由に構成します。

## 外部 HTTPS プロキシへのトラフィック構成 {#configure-traffic-to-external-https-proxy}

1. HTTPS プロキシ用の TCP（HTTP ではありません！）ServiceEntry を定義します。アプリケーションは HTTP CONNECT で HTTPS プロキシと接続しますが、通信自体は TCP なので HTTP ではなく TCP で構成する必要があります。接続確立後、プロキシは単なる TCP トンネルとして動作します。

   {{< text bash >}}
   $ kubectl apply -f - <<EOF
   apiVersion: networking.istio.io/v1
   kind: ServiceEntry
   metadata:
   name: proxy
   spec:
   hosts:

   - my-company-proxy.com # 無視されます
     addresses:
   - $PROXY_IP/32
     ports:
   - number: $PROXY_PORT
     name: tcp
     protocol: TCP
     location: MESH_EXTERNAL
     resolution: NONE
     EOF
     {{< /text >}}

1. `external` 名前空間の curl Pod からリクエストを送信します。curl Pod には Sidecar があるため、Istio でトラフィックを制御できます。

   {{< text bash >}}
   $ kubectl exec -it $SOURCE_POD -c curl -- sh -c "HTTPS_PROXY=$PROXY_IP:$PROXY_PORT curl https://en.wikipedia.org/wiki/Main_Page" | grep -o "<title>.\*</title>"
   <title>Wikipedia, the free encyclopedia</title>
   {{< /text >}}

1. Istio Sidecar プロキシのログを確認します：

   {{< text bash >}}
   $ kubectl logs $SOURCE_POD -c istio-proxy
   [2018-12-07T10:38:02.841Z] "- - -" 0 - 702 87599 92 - "-" "-" "-" "-" "172.30.109.95:3128" outbound|3128||my-company-proxy.com 172.30.230.52:44478 172.30.109.95:3128 172.30.230.52:44476 -
   {{< /text >}}

1. プロキシのアクセスログを確認します：

   {{< text bash >}}
   $ kubectl exec -it $(kubectl get pod -n external -l app=squid -o jsonpath={.items..metadata.name}) -n external -- tail -f /var/log/squid/access.log
   1544160065.248 228 172.30.109.89 TCP_TUNNEL/200 87633 CONNECT en.wikipedia.org:443 - HIER_DIRECT/91.198.174.192 -
   {{< /text >}}

## 仕組みの理解 {#understanding-what-happened}

この例では、次の手順を実施しました：

1. 外部プロキシを模擬するため HTTPS プロキシをデプロイ
1. Istio で制御するトラフィックを外部プロキシに誘導するため TCP ServiceEntry を作成

注意：外部プロキシ経由でアクセスする外部サービス（例：wikipedia.org）に対して ServiceEntry を作成することはできません。Istio から見るとリクエストは外部プロキシにしか送信されず、プロキシがその先にリクエストを転送することは Istio からは見えないためです。

## クリーンアップ {#cleanup}

1. [curl]({{< github_tree >}}/samples/curl) サービスを削除します：

   {{< text bash >}}
   $ kubectl delete -f @samples/curl/curl.yaml@
   {{< /text >}}

1. `external` 名前空間の [curl]({{< github_tree >}}/samples/curl) サービスを削除します：

   {{< text bash >}}
   $ kubectl delete -f @samples/curl/curl.yaml@ -n external
   {{< /text >}}

1. Squid プロキシを停止し、ConfigMap と設定ファイルを削除します：

   {{< text bash >}}
   $ kubectl delete -n external deployment squid
   $ kubectl delete -n external configmap proxy-configmap
   $ rm ./proxy.conf
   {{< /text >}}

1. `external` 名前空間を削除します：

   {{< text bash >}}
   $ kubectl delete namespace external
   {{< /text >}}

1. Service Entry を削除します：

   {{< text bash >}}
   $ kubectl delete serviceentry proxy
   {{< /text >}}
