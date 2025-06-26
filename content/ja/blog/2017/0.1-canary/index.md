---
title: Istio を使用した金絲雀デプロイ
description: Istio を使用して自動スケーリングされた金絲雀デプロイを作成します。
publishdate: 2017-06-14
last_update: 2018-05-16
attribution: Frank Budinsky
keywords: [traffic-management,canary]
aliases:
    - /ja/blog/canary-deployments-using-istio.html
---

{{< tip >}}
このブログは、2018 年 5 月 16 日に更新されました。最新バージョンのトラフィック管理モデルが採用されています。
{{< /tip >}}

[Istio](/ja/) プロジェクトの最大の利点の 1 つは、サービスの金絲雀デプロイを制御するのに便利であることです。金絲雀デプロイ（またはデプロイ）の背後にある考え方は、新しいバージョンをテストするために、小さなユーザー トラフィックの一部を導入し、すべてが順調に進む場合は、パーセンテージを増やす（おそらく徐々に増やす）ことで、古いバージョンを徐々に置き換えることです。問題が発生した場合は、中止して古いバージョンにロールバックできます。最も単純な方法は、ランダムにパーセンテージを選択して金絲雀バージョンにリクエストすることですが、より複雑なシナリオでは、リクエストの地域、ユーザー、またはその他の属性に基づいて、金絲雀バージョンにリクエストをルーティングすることもできます。

基盤の専門知識に基づいて、Istio が金絲雀デプロイをサポートする必要がある理由を疑問に思うかもしれません。Kubernetes などのプラットフォームは、[バージョンのデプロイ](https://kubernetes.io/zh-cn/docs/concepts/workloads/controllers/deployment/#updating-a-deployment) と [金絲雀デプロイ](https://kubernetes.io/zh-cn/docs/concepts/cluster-administration/manage-deployment/#canary-deployments) を行う方法を提供しているからです。
問題は解決しましたか？そうではありません。このようなデプロイは、単純なシナリオでは機能しますが、特に大規模な自動スケーリングのクラウド環境で大きなトラフィックがある場合は、機能が非常に制限されています。

## Kubernetes の金絲雀デプロイ{#canary-deployment-in-Kubernetes}

既にデプロイされている **helloworld** サービスの **v1** バージョンがあり、新しいバージョン **v2** をテスト（または単純にデプロイ）したいとします。
Kubernetes を使用すると、サービスの [Deployment](https://kubernetes.io/zh-cn/docs/concepts/workloads/controllers/deployment/)
のイメージを更新し、[デプロイ](https://kubernetes.io/zh-cn/docs/concepts/workloads/controllers/deployment/#updating-a-deployment) することで、新しいバージョンの **helloworld** サービスをデプロイできます。

もし、[一時停止](https://kubernetes.io/zh-cn/docs/concepts/workloads/controllers/deployment/#pausing-and-resuming-a-deployment) という方法で、v2 の 1 つまたは 2 つの Pod を起動し、その後、**v1** の Pod が十分に実行されていることを確認することができれば、金絲雀デプロイはシステムへの影響を最小限に抑えることができます。

その後、結果を観察するか、必要に応じて [ロールバック](https://kubernetes.io/zh-cn/docs/concepts/workloads/controllers/deployment/#rolling-back-a-deployment) を行うことができます。

理想的には、Deployment に [HPA](https://kubernetes.io/ja/docs/concepts/workloads/controllers/deployment/#scaling-a-deployment) を設定し、
トラフィック負荷を処理するためにレプリカを減らしたり増やしたりする際に、レプリカ比率を一貫して保つこともできます。

このような方法は、十分にテストされたバージョンのデプロイに適していますが、つまり、より多くのブルー/グリーンデプロイ（またはレッド/ブラックデプロイ）であり、「蜻蜓点水」のような金絲雀デプロイではありません。
実際、後者（完全に準備ができていないか、外部に公開されていないバージョンなど）の場合、
Kubernetes の金絲雀デプロイは、[パブリック pod ラベル](https://kubernetes.io/ja/docs/concepts/cluster-administration/manage-deployment/#using-labels-effectively) を持つ 2 つの Deployment を使用して実現されます。
この場合、自動スケーラーを使用できなくなります。なぜなら、異なる負荷状況では、必要な比率と異なる可能性があるためです。

どちらのデプロイを使用しても、Docker、Mesos/Marathon または Kubernetes などのコンテナオーケストレーションプラットフォームを使用した金絲雀デプロイ管理には根本的な問題があります。つまり、インスタンスのスケーリングを使用してトラフィックを管理することです。バージョントラフィックの配布とレプリカのデプロイは、上記のプラットフォームでは独立して実行されます。すべての pod レプリカは、`kube-proxy` ループプールで一視同仁に扱われるため、特定のバージョンが受信するトラフィックを管理する唯一の方法は、レプリカ比率を制御することです。小さなパーセンテージで金絲雀トラフィックを維持するには、多くのレプリカが必要です（例えば、1％ では少なくとも 100 個のレプリカが必要です）。この問題を無視できる場合で

## Istio を使用する{#enter-Istio}

Istio を使用すると、トラフィックルーティングとレプリカのデプロイは完全に独立した機能です。サービスの pod 数は、トラフィック負荷に応じて柔軟にスケーリングでき、バージョントラフィックルーティングの制御と完全に直交します。これにより、自動スケーリングの場合でも、金絲雀バージョンをより簡単に管理できます。実際、自動スケーリングマネージャーは引き続き独立して実行され、トラフィックルーティングによる負荷変化に対する応答と、他の原因による負荷変化に対する応答に違いはありません。

Istio の[ルーティングルール](/ja/docs/concepts/traffic-management/#routing-rules)も便利な機能です。細かい粒度でトラフィックパーセンテージを制御することができます（例えば、100 個の Pod を必要なく 1％ のトラフィックをルーティングすることができます）。また、他のルールを使用してトラフィックを制御することもできます（例えば、特定のユーザーのトラフィックを金絲雀バージョンにルーティングすることができます）。これを示すために、**helloworld** サービスをこの方法でデプロイする簡単な例を見てみましょう。

まず、**helloworld** サービスを定義します。これは、通常の **Kubernetes** サービスと同じです。

{{< text yaml >}}
apiVersion: v1
kind: Service
metadata:
name: helloworld
labels:
  app: helloworld
spec:
  selector:
    app: helloworld
  ...
{{< /text >}}

次に、バージョン **v1** と **v2** の 2 つの Deployment を追加します。これらのバージョンは、サービス選択ラベル `app: helloworld` を含んでいます。

{{< text yaml >}}
kind: Deployment
metadata:
  name: helloworld-v1
spec:
  replicas: 1
  template:
    metadata:
      labels:
        app: helloworld
        version: v1
    spec:
      containers:
      - image: helloworld-v1
        ...
---
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: helloworld-v2
spec:
  replicas: 1
  template:
    metadata:
      labels:
        app: helloworld
        version: v2
    spec:
      containers:
      - image: helloworld-v2
        ...
{{< /text >}}

これは、通常の Kubernetes を使用した[金絲雀デプロイ](https://kubernetes.io/ja/docs/concepts/cluster-administration/manage-deployment/#canary-deployments)と同じですが、Kubernetes 方式では、各 Deployment のレプリカ数を調整してトラフィック分配を制御する必要があります。例えば、金絲雀バージョン（v2）に 10％ のトラフィックを送信する場合、v1 と v2 のレプリカはそれぞれ 9 と 1 に設定できます。

しかし、[Istio を有効にしたクラスター](/ja/docs/setup/)では、ルーティングルールを設定してトラフィック分配を制御できます。例えば、金絲雀バージョンに 10％ のトラフィックを送信する場合、`kubectl` を使用して以下のルーティングルールを設定できます。

{{< text bash >}}
$ kubectl apply -f - <<EOF
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: helloworld
spec:
  hosts:
    - helloworld
  http:
  - route:
    - destination:
        host: helloworld
        subset: v1
        weight: 90
    - destination:
        host: helloworld
        subset: v2
        weight: 10
---
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: helloworld
spec:
  host: helloworld
  subsets:
  - name: v1
    labels:
      version: v1
  - name: v2
    labels:
      version: v2
EOF
{{< /text >}}

ルーティングルールが有効になると、Istio は、各バージョンの実行中のレプリカ数がいくつであっても、金絲雀バージョンに 10％ のリクエストのみを送信することを保証します。

## デプロイメントの自動スケーリング{#autoscaling-the-deployments}

レプリカ比率を維持する必要がなくなったため、Kubernetes [HPA](https://kubernetes.io/ja/docs/tasks/run-application/horizontal-pod-autoscale/) を使用して、2 つのバージョンの Deployment のレプリカを管理できます。

{{< text bash >}}
$ kubectl autoscale deployment helloworld-v1 --cpu-percent=50 --min=1 --max=10
deployment "helloworld-v1" autoscaled
{{< /text >}}

{{< text bash >}}
$ kubectl autoscale deployment helloworld-v2 --cpu-percent=50 --min=1 --max=10
deployment "helloworld-v2" autoscaled
{{< /text >}}

{{< text bash >}}
$ kubectl get hpa
NAME           REFERENCE                 TARGET  CURRENT  MINPODS  MAXPODS  AGE
Helloworld-v1  Deployment/helloworld-v1  50%     47%      1        10       17s
Helloworld-v2  Deployment/helloworld-v2  50%     40%      1        10       15s
{{< /text >}}

現在、**helloworld** サービスに負荷がかかると、**v1** のレプリカ数が **v2** よりもはるかに多くなることがわかります。なぜなら、**v1** pod が 90％ の負荷を処理しているからです。

{{< text bash >}}
$ kubectl get pods | grep helloworld
helloworld-v1-3523621687-3q5wh   0/2       Pending   0          15m
helloworld-v1-3523621687-73642   2/2       Running   0          11m
helloworld-v1-3523621687-7hs31   2/2       Running   0          19m
helloworld-v1-3523621687-dt7n7   2/2       Running   0          50m
helloworld-v1-3523621687-gdhq9   2/2       Running   0          11m
helloworld-v1-3523621687-jxs4t   0/2       Pending   0          15m
helloworld-v1-3523621687-l8rjn   2/2       Running   0          19m
helloworld-v1-3523621687-wwddw   2/2       Running   0          15m
helloworld-v1-3523621687-xlt26   0/2       Pending   0          19m
helloworld-v2-4095161145-963wt   2/2       Running   0          50m
{{< /text >}}

ルーティングルールを変更して **v2** に 50％ のトラフィックを送信すると、短い遅延後に **v1** のレプリカ数が減少し、**v2** のレプリカ数が増加することがわかります。

{{< text bash >}}
$ kubectl get pods | grep helloworld
helloworld-v1-3523621687-73642   2/2       Running   0          35m
helloworld-v1-3523621687-7hs31   2/2       Running   0          43m
helloworld-v1-3523621687-dt7n7   2/2       Running   0          1h
helloworld-v1-3523621687-gdhq9   2/2       Running   0          35m
helloworld-v1-3523621687-l8rjn   2/2       Running   0          43m
helloworld-v2-4095161145-57537   0/2       Pending   0          21m
helloworld-v2-4095161145-9322m   2/2       Running   0          21m
helloworld-v2-4095161145-963wt   2/2       Running   0          1h
helloworld-v2-4095161145-c3dpj   0/2       Pending   0          21m
helloworld-v2-4095161145-t2ccm   0/2       Pending   0          17m
helloworld-v2-4095161145-v3v9n   0/2       Pending   0          13m
{{< /text >}}

最終的な結果は、Kubernetes Deployment のデプロイと非常に似ていますが、全体のプロセスが集中管理されていないことが異なります。それにもかかわらず、いくつかのコンポーネントが独立して動作していることがわかります。

違いは、負荷を停止したときに、ルーティングルールを設定しても、2 つのバージョンのレプリカ数が最終的に最小値（1）になることです。

{{< text bash >}}
$ kubectl get pods | grep helloworld
helloworld-v1-3523621687-dt7n7   2/2       Running   0          1h
helloworld-v2-4095161145-963wt   2/2       Running   0          1h
{{< /text >}}

## 焦点を当てた金絲雀テスト{#focused-canary-testing}

上記のように、Istio ルーティングルールは、特定のルールに基づいてトラフィックをルーティングするために使用できます。これにより、より複雑な金絲雀デプロイメントを提供できます。例えば、シンプルに金絲雀バージョンを任意のパーセンテージのユーザーに公開する方法とは異なり、内部ユーザーに対してテストを行うことができます。

以下のコマンドは、特定のウェブサイトの 50％ のユーザートラフィックを金絲雀バージョンにルーティングし、他のユーザーには影響を与えません。

{{< text bash >}}
$ kubectl apply -f - <<EOF
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: helloworld
spec:
  hosts:
    - helloworld
  http:
  - match:
    - headers:
        cookie:
          regex: "^(.*?;)?(email=[^;]*@some-company-name.com)(;.*)?$"
    route:
    - destination:
        host: helloworld
        subset: v1
        weight: 50
    - destination:
        host: helloworld
        subset: v2
        weight: 50
  - route:
    - destination:
        host: helloworld
        subset: v1
EOF
{{< /text >}}

以前と同様に、2 つのバージョンの Deployment にバインドされた自動スケーラーは、トラフィック分配に影響を与えずに、レプリカを自動的に管理します。

## まとめ{#summary}

この記事では、Istio がどのように汎用の金絲雀デプロイをサポートし、Kubernetes デプロイとの違いを示しました。Istio サービスメッシュは、トラフィック分配を管理するための基本的な制御を提供し、デプロイメントのスケーリングとは完全に独立しています。これにより、金絲雀テストとデプロイを簡単かつ強力な方法で行うことができます。

金絲雀デプロイをサポートするスマートルーティングは、Istio の機能の 1 つにすぎません。これにより、大規模なマイクロサービスベースのアプリケーションの本番環境へのデプロイがより簡単になります。詳細については、[istio.io](/ja/) を参照してください。

サンプルコードは[こちら]({{<github_tree>}}/samples/helloworld)でご確認いただけます。
