---
title: Istio Ingress Gateway の設定
overview: Ingress からトラフィック制御を始める。
weight: 71

owner: istio/wg-docs-maintainers
test: no
---

これまで、Kubernetes Ingress を使って外部からアプリケーションにアクセスしてきました。このモジュールでは、Istio Ingress Gateway を使ってトラフィックを制御し、Istio を利用したマイクロサービス間のトラフィック制御を体験します。

1. 名前空間 `NAMESPACE` を環境変数に保存します。ログでマイクロサービスを識別する際に必要です。

   {{< text bash >}}
   $ export NAMESPACE=$(kubectl config view -o jsonpath="{.contexts[?(@.name == \"$(kubectl config current-context)\")].context.namespace}")
   $ echo $NAMESPACE
   tutorial
   {{< /text >}}

1. Istio Ingress Gateway のホスト名を環境変数に設定します。

   {{< text bash >}}
   $ export MY_INGRESS_GATEWAY_HOST=istio.$NAMESPACE.bookinfo.com
   $ echo $MY_INGRESS_GATEWAY_HOST
   istio.tutorial.bookinfo.com
   {{< /text >}}

1. Istio Ingress Gateway を設定します：

   {{< text bash >}}
   $ kubectl apply -f - <<EOF
   apiVersion: networking.istio.io/v1
   kind: Gateway
   metadata:
   name: bookinfo-gateway
   spec:
   selector:
   istio: ingressgateway # Istio デフォルトの gateway 実装を使用
   servers:

   - port:
     number: 80
     name: http
     protocol: HTTP
     hosts:
     - $MY_INGRESS_GATEWAY_HOST

   ***

   apiVersion: networking.istio.io/v1
   kind: VirtualService
   metadata:
   name: bookinfo
   spec:
   hosts:

   - $MY_INGRESS_GATEWAY_HOST
     gateways:
   - bookinfo-gateway.$NAMESPACE.svc.cluster.local
     http:
   - match: - uri:
     exact: /productpage - uri:
     exact: /login - uri:
     exact: /logout - uri:
     prefix: /static
     route: - destination:
     host: productpage
     port:
     number: 9080
     EOF
     {{< /text >}}

1. [Ingress の IP とポートの確認](/ja/docs/tasks/traffic-management/ingress/ingress-control/#determining-the-ingress-ip-and-ports)の手順で `INGRESS_HOST` と `INGRESS_PORT` を設定してください。

1. 次のコマンドの出力を `/etc/hosts` ファイルに追加します。

   {{< text bash >}}
   $ echo $INGRESS_HOST $MY_INGRESS_GATEWAY_HOST
   {{< /text >}}

1. コマンドラインからアプリのホームページにアクセスします：

   {{< text bash >}}
   $ curl -s $MY_INGRESS_GATEWAY_HOST:$INGRESS_PORT/productpage | grep -o "<title>.\*</title>"
   <title>Simple Bookstore App</title>
   {{< /text >}}

1. 次のコマンドの出力をブラウザのアドレスバーに貼り付けます：

   {{< text bash >}}
   $ echo http://$MY_INGRESS_GATEWAY_HOST:$INGRESS_PORT/productpage
   {{< /text >}}

1. 新しいターミナルウィンドウで無限ループを設定し、現実世界のユーザートラフィックをシミュレートしてアプリにアクセスします。

   {{< text bash >}}
   $ while :; do curl -s <前のコマンドの出力> | grep -o "<title>.\*</title>"; sleep 1; done
   <title>Simple Bookstore App</title>
   <title>Simple Bookstore App</title>
   <title>Simple Bookstore App</title>
   <title>Simple Bookstore App</title>
   ...
   {{< /text >}}

1. Kiali コンソール `my-kiali.io/kiali/console` の Graph で自分の名前空間を確認します（この `my-kiali.io` URL は、[以前の設定](/ja/docs/examples/microservices-istio/bookinfo-kubernetes/#update-your-etc-hosts-configuration-file)で `/etc/hosts` に追加したものです）。

   ここでは 2 つのトラフィックソースが見えます。1 つは `unknown`（Kubernetes Ingress）、もう 1 つは `istio-ingressgateway istio-system`（Istio Ingress Gateway）です。

   {{< image width="80%"
       link="kiali-ingress-gateway.png"
       caption="Kiali Graph タブ（Istio Ingress Gateway あり）"
       >}}

1. ここで Kubernetes Ingress へのリクエスト送信を停止し、Istio Ingress Gateway のみを使います。
   以前設定した無限ループを停止してください（ターミナルで `Ctrl-C`）。本番環境では、アプリの DNS エントリを Istio ingress gateway の IP に更新するか、外部ロードバランサーを設定する必要があります。

1. Kubernetes Ingress リソースを削除します：

   {{< text bash >}}
   $ kubectl delete ingress bookinfo
   ingress.extensions "bookinfo" deleted
   {{< /text >}}

1. 新しいターミナルウィンドウで、前述の手順に従い、再度現実世界のユーザートラフィックをシミュレートしてください。

1. Kiali コンソールで Graph を確認します。Istio Ingress Gateway がアプリの唯一のトラフィックソースになっているはずです。

   {{< image width="80%"
       link="kiali-ingress-gateway-only.png"
       caption="Kiali Graph タブ（Istio Ingress Gateway のみがトラフィックソース）"
       >}}

これで [Istio ログの設定](/ja/docs/examples/microservices-istio/logs-istio) の準備ができました。
