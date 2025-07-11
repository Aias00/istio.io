---
title: productpage で Istio を有効化
overview: 1 つのマイクロサービス上で Istio コントロールプレーンをデプロイし、Istio を有効化します。
weight: 60
owner: istio/wg-docs-maintainers
test: no
---

前のモジュールで見たように、Istio は Kubernetes の機能を強化し、マイクロサービスの運用をより効率的にします。

このモジュールでは、`productpage` マイクロサービスで Istio を有効化します。
アプリケーションの他の部分はそのまま動作し続けます。Istio は 1 つずつマイクロサービス単位で段階的に有効化できます。
Istio の有効化はマイクロサービスに非侵襲的で、マイクロサービスのコードを変更したりアプリケーションを壊したりすることなく、アプリは継続して稼働しユーザーリクエストに応答できます。

1. デフォルトの目標ルールを適用します：

   {{< text bash >}}
   $ kubectl apply -f {{< github_file >}}/samples/bookinfo/networking/destination-rule-all.yaml
   {{< /text >}}

1. `productpage` マイクロサービスを再デプロイし、Istio を有効化します：

   {{< tip >}}
   このチュートリアルでは、手動で Sidecar を注入する手順を示しています。これはサービスごとに Istio を段階的に有効化する方法を学ぶためです。
   本番環境では[自動 Sidecar 注入](/ja/docs/setup/additional-setup/sidecar-injection/#automatic-sidecar-injection)が推奨されます。
   {{< /tip >}}

   {{< text bash >}}
   $ curl -s {{< github_file >}}/samples/bookinfo/platform/kube/bookinfo.yaml | istioctl kube-inject -f - | sed 's/replicas: 1/replicas: 3/g' | kubectl apply -l app=productpage,version=v1 -f -
   deployment.apps/productpage-v1 configured
   {{< /text >}}

1. アプリケーションの Web ページにアクセスし、アプリが動作していることを確認します。Istio は元のアプリケーションコードを変更せずに追加されています。

1. `productpage` の Pod を確認し、各レプリカに 2 つのコンテナがあることを確認します。1 つ目はマイクロサービス本体、2 つ目は Sidecar プロキシです：

   {{< text bash >}}
   $ kubectl get pods
   details-v1-68868454f5-8nbjv 1/1 Running 0 7h
   details-v1-68868454f5-nmngq 1/1 Running 0 7h
   details-v1-68868454f5-zmj7j 1/1 Running 0 7h
   productpage-v1-6dcdf77948-6tcbf 2/2 Running 0 7h
   productpage-v1-6dcdf77948-t9t97 2/2 Running 0 7h
   productpage-v1-6dcdf77948-tjq5d 2/2 Running 0 7h
   ratings-v1-76f4c9765f-khlvv 1/1 Running 0 7h
   ratings-v1-76f4c9765f-ntvkx 1/1 Running 0 7h
   ratings-v1-76f4c9765f-zd5mp 1/1 Running 0 7h
   reviews-v2-56f6855586-cnrjp 1/1 Running 0 7h
   reviews-v2-56f6855586-lxc49 1/1 Running 0 7h
   reviews-v2-56f6855586-qh84k 1/1 Running 0 7h
   curl-88ddbcfdd-cc85s 1/1 Running 0 7h
   {{< /text >}}

1. Kubernetes は非侵襲的かつ段階的な[ローリングアップデート](https://kubernetes.io/ja/docs/tutorials/kubernetes-basics/update/update-intro/)で、Istio 有効化済みの Pod で元の Pod を置き換えます。Kubernetes は新しい Pod が稼働するまで古い Pod を終了せず、トラフィックを 1 つずつ新しい Pod に切り替えます。つまり、新しい Pod を宣言する前に 1 つ以上の Pod を終了することはありません。これらの操作はアプリケーションの可用性を損なわないためであり、Istio 注入中もアプリは継続して動作します。

1. `productpage` の Istio Sidecar のログを確認します：

   {{< text bash >}}
   $ kubectl logs -l app=productpage -c istio-proxy | grep GET
   ...
   [2019-02-15T09:06:04.079Z] "GET /details/0 HTTP/1.1" 200 - 0 178 5 3 "-" "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_14) AppleWebKit/605.1.15 (KHTML, like Gecko) Version/12.0 Safari/605.1.15" "18710783-58a1-9e5f-992c-9ceff05b74c5" "details:9080" "172.30.230.51:9080" outbound|9080||details.tutorial.svc.cluster.local - 172.21.109.216:9080 172.30.146.104:58698 -
   [2019-02-15T09:06:04.088Z] "GET /reviews/0 HTTP/1.1" 200 - 0 379 22 22 "-" "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_14) AppleWebKit/605.1.15 (KHTML, like Gecko) Version/12.0 Safari/605.1.15" "18710783-58a1-9e5f-992c-9ceff05b74c5" "reviews:9080" "172.30.230.27:9080" outbound|9080||reviews.tutorial.svc.cluster.local - 172.21.185.48:9080 172.30.146.104:41442 -
   [2019-02-15T09:06:04.053Z] "GET /productpage HTTP/1.1" 200 - 0 5723 90 83 "10.127.220.66" "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_14) AppleWebKit/605.1.15 (KHTML, like Gecko) Version/12.0 Safari/605.1.15" "18710783-58a1-9e5f-992c-9ceff05b74c5" "tutorial.bookinfo.com" "127.0.0.1:9080" inbound|9080|http|productpage.tutorial.svc.cluster.local - 172.30.146.104:9080 10.127.220.66:0 -
   {{< /text >}}

1. 名前空間を出力します。Istio ダッシュボードでマイクロサービスを識別する際に使います：

   {{< text bash >}}
   $ echo $(kubectl config view -o jsonpath="{.contexts[?(@.name == \"$(kubectl config current-context)\")].context.namespace}")
   tutorial
   {{< /text >}}

1. Istio ダッシュボードを確認します。カスタム URL は[以前に設定した](/ja/docs/examples/microservices-istio/bookinfo-kubernetes/#update-your-etc-hosts-configuration-file) `/etc/hosts` ファイルに記載されています：

   {{< text plain >}}
   http://my-istio-dashboard.io/dashboard/db/istio-mesh-dashboard
   {{< /text >}}

   左上のドロップダウンメニューで **Istio Mesh Dashboard** を選択します。

   {{< image width="80%"
       link="dashboard-select-dashboard.png"
       caption="左上のドロップダウンメニューで Istio Mesh Dashboard を選択"
       >}}

   名前空間内の `productpage` サービスに注目してください。名前は `productpage.<your namespace>.svc.cluster.local` となっているはずです。

   {{< image width="80%"
       link="dashboard-mesh.png"
       caption="Istio Mesh Dashboard"
       >}}

1. Istio Mesh ダッシュボードの `Service` 列で `productpage` サービスをクリックします。

   {{< image width="80%"
       link="dashboard-service-select-productpage.png"
       caption="Istio Service Dashboard, `productpage` selected"
       >}}

   **Service Workloads** セクションまでスクロールします。ダッシュボードのグラフが更新されていることを確認してください。

   {{< image width="80%"
       link="dashboard-service.png"
       caption="Istio Service Dashboard"
       >}}

これが 1 つのマイクロサービスで Istio を有効化する直接的なメリットです。マイクロサービスの入出力トラフィックのログ（時刻、HTTP メソッド、パス、レスポンスコードなど）を受け取ることができます。Istio ダッシュボードでマイクロサービスを監視できます。

次のモジュールでは、Istio がアプリケーションに提供できる機能について学びます。Istio の機能がマイクロサービスに有益な場合、アプリ全体で Istio を有効化し、その全機能を活用する方法を学びます。

これで[すべてのマイクロサービスで Istio を有効化](/ja/docs/examples/microservices-istio/enable-istio-all-microservices)する準備ができました。
