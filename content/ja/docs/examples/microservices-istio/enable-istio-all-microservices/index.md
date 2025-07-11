---
title: すべてのマイクロサービスで Istio を有効化
overview: アプリ全体で Istio を有効化します。
weight: 70

owner: istio/wg-docs-maintainers
test: no
---

前のセクションでは、`productpage` という単一のマイクロサービスで Istio を有効化しました。
Istio は他のマイクロサービスにも段階的に有効化でき、より多くのマイクロサービスに Istio の機能を追加できます。
このチュートリアルでは、残りのすべてのマイクロサービスで一度に Istio を有効化する方法を説明します。

1.  このチュートリアルの目的のため、まずマイクロサービスのデプロイ数を 1 に縮小します：

    {{< text bash >}}
    $ kubectl scale deployments --all --replicas 1
    {{< /text >}}

1.  Istio を有効化した Bookinfo アプリを再デプロイします。`productpage` サービスはすでに Istio が注入されているため再デプロイされませんし、Pod の変更も不要です。
    この時点で、単一レプリカのマイクロサービスクラスタで Istio を有効化できます。

        {{< text bash >}}
        $ curl -s {{< github_file >}}/samples/bookinfo/platform/kube/bookinfo.yaml | istioctl kube-inject -f - | kubectl apply -l app!=reviews -f -
        $ curl -s {{< github_file >}}/samples/bookinfo/platform/kube/bookinfo.yaml | istioctl kube-inject -f - | kubectl apply -l app=reviews,version=v2 -f -
        service/details unchanged
        serviceaccount/bookinfo-details unchanged
        deployment.apps/details-v1 configured
        service/ratings unchanged
        serviceaccount/bookinfo-ratings unchanged
        deployment.apps/ratings-v1 configured
        serviceaccount/bookinfo-reviews unchanged
        service/productpage unchanged
        serviceaccount/bookinfo-productpage unchanged
        deployment.apps/productpage-v1 configured
        deployment.apps/reviews-v2 configured
        {{< /text >}}

1.  アプリの Web ページに何度かアクセスしてみましょう。Istio の追加は**非侵襲的**であり、
    元のアプリケーションには変更がありません。Istio はアプリの稼働中に追加され、
    アプリ全体のデプロイ解除や再デプロイは不要です。

1.  アプリの Pod を確認し、各 Pod に 2 つのコンテナがあることを検証します。
    1 つはマイクロサービス本体、もう 1 つはマイクロサービスに付加された Sidecar プロキシです：

        {{< text bash >}}
        $ kubectl get pods
        details-v1-58c68b9ff-kz9lf        2/2       Running   0          2m
        productpage-v1-59b4f9f8d5-d4prx   2/2       Running   0          2m
        ratings-v1-b7b7fbbc9-sggxf        2/2       Running   0          2m
        reviews-v2-dfbcf859c-27dvk        2/2       Running   0          2m
        curl-88ddbcfdd-cc85s              1/1       Running   0          7h
        {{< /text >}}

1.  以前[/etc/hosts 設定ファイルの更新](/ja/docs/examples/microservices-istio/bookinfo-kubernetes/#update-your-etc-hosts-configuration-file)で設定したカスタム URL から Istio ダッシュボードにアクセスします：

    {{< text plain >}}
    http://my-istio-dashboard.io/dashboard/db/istio-mesh-dashboard
    {{< /text >}}

1.  左上のドロップダウンメニューで **Istio Mesh Dashboard** を選択します。
    これで、あなたの名前空間内のすべてのサービスがサービスリストに表示されることに注目してください。

        {{< image width="80%"
            link="dashboard-mesh-all.png"
            caption="Istio Mesh Dashboard"
            >}}

1.  **Istio Service Dashboard** ダッシュボードで、`ratings` など他のマイクロサービスも確認できます：

    {{< image width="80%"
        link="dashboard-ratings.png"
        caption="Istio Service Dashboard"
        >}}

1.  [Kiali](https://www.kiali.io) コンソールの可視化 UI でアプリのトポロジーを確認します。
    このコンソールは Istio の一部ではなく、`demo` 設定でインストールされます。
    以前[/etc/hosts 設定ファイルの更新](/ja/docs/examples/microservices-istio/bookinfo-kubernetes/#update-your-etc-hosts-configuration-file)で設定したカスタム URL からこのダッシュボードにアクセスします：

        {{< text plain >}}
        http://my-kiali.io/kiali/console
        {{< /text >}}

        Kiali を[はじめにガイド](/ja/docs/setup/getting-started/)でインストールした場合、
        Kiali コンソールのユーザー名は `admin`、パスワードも `admin` です。

1.  左上で `Graph` タブをクリックし、**Namespace** ドロップダウンで自分の名前空間を選択します。
    **Display** ドロップダウンで **Traffic Animation** チェックボックスをオンにすると、
    クールなトラフィックアニメーションが見られます。

        {{< image width="80%"
            link="kiali-display-menu.png"
            caption="Kiali Graph タブと Display ドロップダウンメニュー"
            >}}

1.  **Edge Labels** ドロップダウンでさまざまな項目を選択してみてください。
    グラフのノードやエッジにマウスを重ねると、右側にトラフィック指標が表示されます。

        {{< image width="80%"
            link="kiali-edge-labels-menu.png"
            caption="Kiali Graph タブと edge labels ドロップダウンメニュー"
            >}}

        {{< image width="80%"
            link="kiali-initial.png"
            caption="Kiali Graph タブ"
            >}}

これで[Istio Ingress Gateway の設定](/ja/docs/examples/microservices-istio/istio-ingress-gateway)に進むことができます。
