---
title: Istio とネットワークポリシー
description: Istio のポリシーが Kubernetes のネットワークポリシーとどのように関連するかを説明します。
publishdate: 2017-08-10
subtitle:
attribution: Spike Curtis
aliases:
    - /ja/blog/using-network-policy-in-concert-with-istio.html
target_release: 0.1
---

Kubernetes 上のアプリケーションを保護するためのネットワークポリシーは、現在広く受け入れられている業界のベストプラクティスです。Istio もポリシーをサポートしているため、Istio ポリシーと Kubernetes ネットワークポリシーの相互作用と、アプリケーションのセキュリティを提供する方法について説明します。

まずは基本から：なぜ Istio と Kubernetes ネットワークポリシーを同時に使用したいのでしょうか？簡単に言うと、それらは異なることを扱うからです。Istio とネットワークポリシーの主な違いを表にまとめました（典型的な実装（Calico など）を説明しますが、具体的な実装の詳細は異なるネットワークプロバイダーによって異なります）：

|                      | Istio ポリシー        |ネットワークポリシー           |
| -------------------- | ----------------- | ------------------ |
| **レイヤー**              |"サービス" --- L7     |"ネットワーク" --- L3-4    |
| **実装**              |ユーザー空間          |カーネル               |
| **実行ポイント**            |Pod               |ノード               |

## レイヤー{#layer}

OSI モデルの観点から 7 レイヤー（アプリケーション）を見ると、Istio ポリシーはネットワークアプリケーションの「サービス」レイヤーで実行されます。しかし、実際のクラウドネイティブアプリケーションモデルは 7 レイヤーには少なくとも 2 つのレイヤーが含まれています：サービスレイヤーとコンテンツレイヤーです。サービスレイヤーは通常 HTTP で、実際のアプリケーションデータ（コンテンツレイヤー）をラップしています。Istio の Envoy 代理は HTTP サービスレイヤーで実行されます。一方、ネットワークポリシーは OSI モデルの第 3 レイヤー（ネットワーク）と第 4 レイヤー（トランスポート）で実行されます。

サービスレイヤーで Envoy 代理が実行されることで、基礎的なプロトコルによるポリシー決定に必要な豊富な属性を提供します。これには HTTP/1.1 や HTTP/2（gRPC は HTTP/2 上で実行されます）が含まれます。したがって、仮想ホスト、URL、または HTTP ヘッダーに基づいてポリシーを適用できます。将来、Istio は広範囲の 7 レイヤーのプロトコルと、一般的な TCP および UDP トランスポートをサポートします。

Istio ポリシーはネットワークレイヤーで実行されることで、すべてのネットワークアプリケーションが IP を使用するため、汎用的な利点があります。7 レイヤーのプロトコルに関係なく、ネットワークレイヤーでポリシーを適用できます：DNS、SQL データベース、リアルタイムストリーミング、および HTTP 以外の多くのサービスを保護できます。ネットワークポリシーは、IP アドレス、プロトコル、およびポートの三つ組みに限定されるのではなく、Istio とネットワークポリシーは、Pod エンドポイントを記述するために豊富な Kubernetes ラベルを使用できます。

## 実装{#implementation}

Istio の代理は {{<gloss envoy>}}Envoy{{</gloss>}} に基づいており、データプレーンのユーザー空間デーモンとして実装されています。これにより、ネットワークレイヤーとの標準的なソケットを使用して、柔軟性が高く、コンテナに配布（およびアップグレード！）できます。

ネットワークポリシーのデータプレーンは通常、カーネル空間で実装されます（例：iptables、

## 実行ポイント{#enforcement-point}

Envoy 代理のポリシー実行は Pod 内で行われ、同一ネットワーク名前空間内のサイドカー容器として実行されます。これにより、デプロイメントモデルがシンプルになります。特定のコンテナには、Pod 内のネットワークを再設定する権限（`CAP_NET_ADMIN`）が付与されています。このようなサービスインスタンスが代理を回避して損傷したり、不適切な動作を示した場合（例：悪意のあるテナント内）。

これは、攻撃者が他の Istio が有効な Pod にアクセスできないようにするわけではありませんが、設定によっては、以下のような攻撃が可能になります：

- 未保護の Pod を攻撃
- 保護された Pod に大量のトラフィックを送信してアクセス拒否を引き起こす
- Pod 内で収集された漏洩データ
- クラスターインフラストラクチャ（サーバーまたは Kubernetes サービス）を攻撃
- データベース、ストレージアレイ、またはレガシーシステムなどのグリッド外のサービスを攻撃

ネットワークポリシーは通常、クライアントのネットワーク名前空間外のホストノードで実行されます。これは、損傷したり、不適切な動作を示した Pod がルート名前空間に入るのを防ぐ必要があることを意味します。Kubernetes 1.8 で egress ポリシーが追加されることで、ネットワークポリシーがクラスターインフラストラクチャを保護するための重要な部分となりました。

## 例{#examples}

Istio アプリケーションが Kubernetes ネットワークポリシーを使用する例を見てみましょう。以下は、ネットワークポリシー機能の使用例として、Bookinfo アプリケーションを例に挙げています：

- アプリケーションエントランスの攻撃面を減らす
- アプリケーション内で細粒度の隔離を実現する

### アプリケーションエントランスの攻撃面を減らす{#reduce-attack-surface-of-the-application-ingress}

アプリケーションの ingress コントローラーは、外部世界からアプリケーションに入る主な入口です。`istio.yaml` （Istio のインストールに使用）で定義されている Istio-ingress を素早く確認してみましょう：

{{< text yaml >}}
apiVersion: v1
kind: Service
metadata:
  name: istio-ingress
  labels:
    istio: ingress
spec:
  type: LoadBalancer
  ports:
  - port: 80
    name: http
  - port: 443
    name: https
  selector:
    istio: ingress
{{< /text >}}

`istio-ingress` は 80 と 443 のポートを公開しています。流入トラフィックをこれらのポートに制限する必要があります。Envoy には [`組み込み管理インターフェース`](https://www.envoyproxy.io/docs/envoy/latest/operations/admin.html#operations-admin-interface) がありますが、`istio-ingress` の誤った設定により、管理インターフェースが外部に公開されることを防ぎたいと思います。ここでは、深度防御の例を示します：正しく設定されたイメージはインターフェースを公開する必要があります。正しく設定されたネットワークポリシーは、誰もそれに接続できないようにします。それは失敗するか、設定が間違っているか、保護されます。

{{< text yaml >}}
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: istio-ingress-lockdown
  namespace: default
spec:
  podSelector:
    matchLabels:
      istio: ingress
  ingress:
  - ports:
    - protocol: TCP
      port: 80
    - protocol: TCP
      port: 443
{{< /text >}}

### アプリケーション内で細粒度の隔離を実現する{#enforce-fine-grained-isolation-within-the-application}

以下は、Bookinfo アプリケーションのサービスの示意図です：

{{< image width="80%"
    link="/zh/docs/examples/bookinfo/withistio.svg"
    caption="Bookinfo Service Graph"
    >}}

この図は、正しく機能するアプリケーションが許可する各接続を示しています。他のすべての接続（例：Istio Ingress から Rating サービスへの接続）は、アプリケーションの一部ではありません。関係のない接続を除外しましょう。例えば：Ingress Pod が攻撃者によって攻撃され、任意のコードを実行することを許可されたとします。もし、ネットワークポリシーを使用して `productpage`（`http://$GATEWAY_URL/productpage`）に接続する Pod のみを許可する場合、攻撃者はアプリケーションのバックエンドにアクセスできなくなります。ただし、それらはすでにサービスメッシュのメンバーを破壊しています。

{{< text yaml >}}
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: product-page-ingress
  namespace: default
spec:
  podSelector:
    matchLabels:
      app: productpage
  ingress:
  - ports:
    - protocol: TCP
      port: 9080
    from:
    - podSelector:
        matchLabels:
          istio: ingress
{{< /text >}}

お勧めは、他の Pod が実行にアクセスできるようにするために、各サービスに対して同様のポリシーを書くことです。

## まとめ{#summary}

Istio とネットワークポリシーは、アプリケーションポリシーの実現に異なる利点があると考えています。Istio はアプリケーションプロトコル認識と高い柔軟性を備えており、サービスルーティング、再試行、フォールバックなどの運用目標をサポートするためのアプリケーションポリシーに適しています。また、アプリケーション層でのセキュリティを有効にすることもできます。例えば、トークンの検証などです。ネットワークポリシーは、汎用的で、効率的で、Pod を隔離するため、アプリケーションポリシーをサポートするための理想的な選択肢です。さらに、ネットワークスタックの異なるレイヤーで実行されるポリシーは、状態を混合せず、責任分離を可能にするため、非常に良いことです。

この記事は、Spike Curtis の三部作のブログシリーズに基づいています。彼は Tigera の Istio チームのメンバーです。完全なシリーズはこちらで見ることができます：<https://www.projectcalico.org/using-network-policy-in-concert-with-istio/>
