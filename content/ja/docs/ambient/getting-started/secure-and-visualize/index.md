---
title: アプリケーションの保護と可視化
description: Ambient モードを有効にしてアプリケーション間の通信を保護します。
weight: 3
owner: istio/wg-networking-maintainers
test: yes
---

アプリケーションを Ambient メッシュに追加するのは、アプリケーションが配置されている名前空間にラベルを付けるのと同じくらい簡単です。
アプリケーションをメッシュに追加すると、Istio は自動的にそれらの間の通信を保護し、TCP 可観測データを収集します。
さらに、アプリケーションを再起動または再デプロイする必要はありません！

## Bookinfo をメッシュに追加 {#add-bookinfo-to-the-mesh}

名前空間にラベルを付けるだけで、指定された名前空間内のすべての Pod を Ambient メッシュに追加できます：

{{< text bash >}}
$ kubectl label namespace default istio.io/dataplane-mode=ambient
namespace/default labeled
{{< /text >}}

おめでとうございます！デフォルトの名前空間内のすべての Pod が Ambient メッシュに追加されました。🎉

ブラウザで Bookinfo アプリケーションを開くと、前と同様に、製品ページが表示されます。
今回の違いは、Bookinfo アプリケーションの Pod 間の通信が mTLS で暗号化されていることです。
さらに、Istio は Pod 間のすべてのトラフィックの TCP 可観測データを収集します。

{{< tip >}}
すべての Pod 間で mTLS 暗号化が有効になりました - さらに、アプリケーションを再起動または再デプロイする必要はありません！
{{< /tip >}}

## アプリケーションと指標の可視化 {#visualize-the-application-and-metrics}

Istio のダッシュボード、Kiali、および Prometheus 指標エンジンを使用すると、Bookinfo アプリケーションを可視化できます。
それらをデプロイします：

{{< text syntax=bash snip_id=none >}}
$ kubectl apply -f @samples/addons/prometheus.yaml@
$ kubectl apply -f @samples/addons/kiali.yaml@
{{< /text >}}

Kiali ダッシュボードにアクセスするには、以下のコマンドを実行してください：

{{< text syntax=bash snip_id=none >}}
$ istioctl dashboard kiali
{{< /text >}}

Bookinfo アプリケーションにトラフィックを送信して、Kiali がトラフィックマップを生成できるようにします：

{{< text bash >}}
$ for i in $(seq 1 100); do curl -sSI -o /dev/null http://localhost:8080/productpage; done
{{< /text >}}

トラフィックマップをクリックし、"Select Namespaces" ドロップダウンメニューから "Default" を選択します。
Bookinfo アプリケーションが表示されます：

{{< image link="./kiali-ambient-bookinfo.png" caption="Kiali 仪表盘" >}}

{{< tip >}}
トラフィックマップが表示されない場合は、Bookinfo アプリケーションにトラフィックを再送信し、
Kiali の **Namespace** ドロップダウンメニューで **default** 名前空間が選択されていることを確認してください。

サービス間の mTLS 状態を表示するには、**Display** ドロップダウンメニューをクリックし、**Security** をクリックします。
{{</ tip >}}

ダッシュボード上の2つのサービスを接続する線をクリックすると、Istio が収集したインバウンドおよびアウトバウンドトラフィック指標を確認できます。

{{< image link="./kiali-tcp-traffic.png" caption="L4 流量" >}}

TCP 指標以外に、Istio は各サービスに強力な ID を作成します：
SPIFFE ID。この ID は認証ポリシーを作成するために使用できます。

## 下一步 {#next-steps}

サービスに ID を割り当てたら、[認証ポリシーを実行](/zh/docs/ambient/getting-started/enforce-auth-policies/)して、アプリケーションへのアクセスを保護します。
