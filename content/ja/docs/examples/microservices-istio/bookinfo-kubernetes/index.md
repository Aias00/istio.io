---
title: Kubernetes で Bookinfo を実行する
overview: Kubernetes で ratings マイクロサービスを使った Bookinfo アプリケーションをデプロイします。
weight: 30

owner: istio/wg-docs-maintainers
test: no
---

{{< boilerplate work-in-progress >}}

このモジュールでは、4 つの異なるプログラミング言語で書かれたマイクロサービス（`productpage`、`details`、`ratings`、`reviews`）からなるアプリケーションを紹介します。このアプリケーションを「Bookinfo」と呼びます。詳細は [Bookinfo サンプル](/ja/docs/examples/bookinfo)ページをご覧ください。

`reviews` マイクロサービスには 3 つのバージョン（`v1`、`v2`、`v3`）がありますが、[Bookinfo サンプル](/ja/docs/examples/bookinfo)はアプリの最終形を示しています。このモジュールでは、アプリケーションは `reviews` マイクロサービスの `v1` バージョンのみを使用します。以降のモジュールで、`reviews` マイクロサービスの複数バージョンを使ってアプリケーションを拡張していきます。

## アプリケーションとテスト Pod のデプロイ {#deploy-the-application-and-a-testing-pod}

1. 環境変数 `MYHOST` をアプリケーションの URL で設定します：

   {{< text bash >}}
   $ export MYHOST=$(kubectl config view -o jsonpath={.contexts..namespace}).bookinfo.com
   {{< /text >}}

1. [`bookinfo.yaml`]({{< github_blob >}}/samples/bookinfo/platform/kube/bookinfo.yaml) を確認します。これはアプリの Kubernetes デプロイメント仕様です。Service と Deployment に注目してください。

1. アプリケーションを Kubernetes クラスタにデプロイします：

   {{< text bash >}}
   $ kubectl apply -l version!=v2,version!=v3 -f {{< github_file >}}/samples/bookinfo/platform/kube/bookinfo.yaml
   service/details created
   serviceaccount/bookinfo-details created
   deployment.apps/details-v1 created
   service/ratings created
   serviceaccount/bookinfo-ratings created
   deployment.apps/ratings-v1 created
   service/reviews created
   serviceaccount/bookinfo-reviews created
   deployment.apps/reviews-v1 created
   service/productpage created
   serviceaccount/bookinfo-productpage created
   deployment.apps/productpage-v1 created
   {{< /text >}}

1. Pod の状態を確認します：

   {{< text bash >}}
   $ kubectl get pods
   NAME READY STATUS RESTARTS AGE
   details-v1-6d86fd9949-q8rrf 1/1 Running 0 10s
   productpage-v1-c9965499-tjdjx 1/1 Running 0 8s
   ratings-v1-7bf577cb77-pq9kg 1/1 Running 0 9s
   reviews-v1-77c65dc5c6-kjvxs 1/1 Running 0 9s
   {{< /text >}}

1. 4 つのサービスが `Running` 状態になったら、deployment をスケールします。各マイクロサービスの各バージョンを 3 Pod で動かすには、次のコマンドを実行します：

   {{< text bash >}}
   $ kubectl scale deployments --all --replicas 3
   deployment.apps/details-v1 scaled
   deployment.apps/productpage-v1 scaled
   deployment.apps/ratings-v1 scaled
   deployment.apps/reviews-v1 scaled
   {{< /text >}}

1. Pod の状態を確認し、各マイクロサービスに 3 つの Pod があることを確認します：

   {{< text bash >}}
   $ kubectl get pods
   NAME READY STATUS RESTARTS AGE
   details-v1-6d86fd9949-fr59p 1/1 Running 0 50s
   details-v1-6d86fd9949-mksv7 1/1 Running 0 50s
   details-v1-6d86fd9949-q8rrf 1/1 Running 0 1m
   productpage-v1-c9965499-hwhcn 1/1 Running 0 50s
   productpage-v1-c9965499-nccwq 1/1 Running 0 50s
   productpage-v1-c9965499-tjdjx 1/1 Running 0 1m
   ratings-v1-7bf577cb77-cbdsg 1/1 Running 0 50s
   ratings-v1-7bf577cb77-cz6jm 1/1 Running 0 50s
   ratings-v1-7bf577cb77-pq9kg 1/1 Running 0 1m
   reviews-v1-77c65dc5c6-5wt8g 1/1 Running 0 49s
   reviews-v1-77c65dc5c6-kjvxs 1/1 Running 0 1m
   reviews-v1-77c65dc5c6-r55tl 1/1 Running 0 49s
   {{< /text >}}

1. サービスが `Running` 状態になったら、テスト用 Pod（[curl]({{< github_tree >}}/samples/curl)）をデプロイします。この Pod からマイクロサービスにリクエストを送信できます：

   {{< text bash >}}
   $ kubectl apply -f {{< github_file >}}/samples/curl/curl.yaml
   {{< /text >}}

1. テスト Pod から curl コマンドで Bookinfo アプリにリクエストを送り、正常に動作していることを確認します：

   {{< text bash >}}
   $ kubectl exec $(kubectl get pod -l app=curl -o jsonpath='{.items[0].metadata.name}') -c curl -- curl -sS productpage:9080/productpage | grep -o "<title>.\*</title>"
   <title>Simple Bookstore App</title>
   {{< /text >}}

## アプリケーションへの外部アクセスを有効化 {#enable-external-access-to-the-application}

アプリケーションが動作したら、クラスタ外部のクライアントからアクセスできるようにします。以下の手順を正しく設定すれば、ノート PC のブラウザからアプリケーションにアクセスできます。

{{< warning >}}

クラスタが GKE 上で動作している場合は、`productpage` サービスのタイプを `LoadBalancer` に変更してください。例：

{{< text bash >}}
$ kubectl patch svc productpage -p '{"spec": {"type": "LoadBalancer"}}'
service/productpage patched
{{< /text >}}

{{< /warning >}}

### Kubernetes Ingress リソースを設定しアプリページにアクセス {#configure-the-Kubernetes-Ingress-resource-and-access-your-application-webpage}

1. Kubernetes Ingress リソースを作成します：

   {{< text bash >}}
   $ kubectl apply -f - <<EOF
   apiVersion: networking.k8s.io/v1
   kind: Ingress
   metadata:
   name: bookinfo
   annotations:
   kubernetes.io/ingress.class: istio
   spec:
   rules:

   - host: $MYHOST
     http:
     paths: - path: /productpage
     pathType: Prefix
     backend:
     service:
     name: productpage
     port:
     number: 9080 - path: /login
     pathType: Prefix
     backend:
     service:
     name: productpage
     port:
     number: 9080 - path: /logout
     pathType: Prefix
     backend:
     service:
     name: productpage
     port:
     number: 9080 - path: /static
     pathType: Prefix
     backend:
     service:
     name: productpage
     port:
     number: 9080
     EOF
     {{< /text >}}

### `/etc/hosts` 設定ファイルの更新 {#update-your-etc-hosts-configuration-file}

1. `bookinfo` という名前の Kubernetes Ingress の IP アドレスを取得します：

   {{< text bash >}}
   $ kubectl get ingress bookinfo
   {{< /text >}}

1. 次のコマンドの出力を `/etc/hosts` ファイルに追記します。スーパーユーザー権限が必要な場合があり、[`sudo`](https://en.wikipedia.org/wiki/Sudo) を使って `/etc/hosts` を編集してください。

   {{< text bash >}}
   $ echo $(kubectl get ingress istio-system -n istio-system -o jsonpath='{..ip} {..host}') $(kubectl get ingress bookinfo -o jsonpath='{..host}')
   {{< /text >}}

### アプリケーションへのアクセス {#access-your-application}

1. 次のコマンドでアプリのホームページにアクセスします：

   {{< text bash >}}
   $ curl -s $MYHOST/productpage | grep -o "<title>.\*</title>"
   <title>Simple Bookstore App</title>
   {{< /text >}}

1. 次のコマンドの出力をブラウザのアドレスバーに貼り付けます：

   {{< text bash >}}
   $ echo http://$MYHOST/productpage
   {{< /text >}}

   以下のページが表示されます：

   {{< image width="80%"
       link="bookinfo.png"
       caption="Bookinfo Web Application"
       >}}

1. マイクロサービスがどのように相互に呼び出されているかを観察します。たとえば、`reviews` は URL `http://ratings:9080/ratings` で `ratings` マイクロサービスを呼び出します。[`reviews` のコード]({{< github_blob >}}/samples/bookinfo/src/reviews/reviews-application/src/main/java/application/rest/LibertyRestEndpoint.java)を参照してください：

   {{< text java >}}
   private final static String ratings_service = "http://ratings:9080/ratings";
   {{< /text >}}

1. 別のターミナルウィンドウで無限ループを設定し、アプリケーションにトラフィックを送り続けて現実世界のユーザートラフィックをシミュレートします：

   {{< text bash >}}
   $ while :; do curl -s $MYHOST/productpage | grep -o "<title>.\*</title>"; sleep 1; done
   <title>Simple Bookstore App</title>
   <title>Simple Bookstore App</title>
   <title>Simple Bookstore App</title>
   <title>Simple Bookstore App</title>
   ...
   {{< /text >}}

これで[アプリケーションのテスト](/ja/docs/examples/microservices-istio/production-testing)の準備ができました。
