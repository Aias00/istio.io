---
title: サンプルアプリケーションをデプロイ
description: Bookinfo サンプルアプリケーションをデプロイします。
weight: 2
owner: istio/wg-networking-maintainers
test: yes
prev: /docs/ambient/getting-started
---

Istio を探索するには、[Bookinfo アプリケーション](/zh/docs/examples/bookinfo/)をインストールする必要があります。
これは、Istio の機能を示すために 4 つの独立したマイクロサービスで構成されています。

{{< image width="50%" link="./bookinfo.svg" caption="Istio の Bookinfo サンプルアプリケーションは、複数の言語で書かれています" >}}

本ガイドの一部として、Bookinfo アプリケーションをデプロイし、エントリーゲートウェイを使用して `productpage` サービスを公開します。

## Bookinfo アプリケーションをデプロイ {#deploy-the-bookinfo-application}

まず、アプリケーションをデプロイします：

{{< text bash >}}
$ kubectl apply -f @samples/bookinfo/platform/kube/bookinfo.yaml@
$ kubectl apply -f @samples/bookinfo/platform/kube/bookinfo-versions.yaml@
{{< /text >}}

アプリケーションが実行中かどうかを確認するには、Pod の状態を確認します：

{{< text syntax=bash snip_id=none >}}
$ kubectl get pods
NAME                             READY   STATUS    RESTARTS   AGE
details-v1-cf74bb974-nw94k       1/1     Running   0          42s
productpage-v1-87d54dd59-wl7qf   1/1     Running   0          42s
ratings-v1-7c4bbf97db-rwkw5      1/1     Running   0          42s
reviews-v1-5fd6d4f8f8-66j45      1/1     Running   0          42s
reviews-v2-6f9b55c5db-6ts96      1/1     Running   0          42s
reviews-v3-7d99fd7978-dm6mx      1/1     Running   0          42s
{{< /text >}}

クラスター外部から `productpage` サービスにアクセスするには、エントリーゲートウェイを

Kubernetes Gateway API を使用して `bookinfo-gateway` という名前のゲートウェイをデプロイします：

{{< text syntax=bash snip_id=deploy_bookinfo_gateway >}}
$ kubectl apply -f @samples/bookinfo/gateway-api/bookinfo-gateway.yaml@
{{< /text >}}

デフォルトでは、Istio はゲートウェイの `LoadBalancer` サービスを作成します。
ゲートウェイにトンネル経由でアクセスするため、ロードバランサーは不要です。
ゲートウェイのサービスタイプを `ClusterIP` に変更するには、注釈を使用します：

{{< text syntax=bash snip_id=annotate_bookinfo_gateway >}}
$ kubectl annotate gateway bookinfo-gateway networking.istio.io/service-type=ClusterIP --namespace=default
{{< /text >}}

ゲートウェイの状態を確認するには、以下のコマンドを実行してください：

{{< text bash >}}
$ kubectl get gateway
NAME               CLASS   ADDRESS                                            PROGRAMMED   AGE
bookinfo-gateway   istio   bookinfo-gateway-istio.default.svc.cluster.local   True         42s
{{< /text >}}

ゲートウェイがプログラムに従って表示されるのを待ってから続行してください。

## 访问应用程序 {#access-the-application}

あなたは、ちょうど設定したゲートウェイを通じて Bookinfo `productpage` サービスに接続します。
ゲートウェイにアクセスするには、`kubectl port-forward` コマンドを使用してください：

{{< text syntax=bash snip_id=none >}}
$ kubectl port-forward svc/bookinfo-gateway-istio 8080:80
{{< /text >}}

ブラウザを開き、`http://localhost:8080/productpage` に移動して Bookinfo アプリケーションを表示します。

{{< image width="80%" link="./bookinfo-browser.png" caption="Bookinfo アプリケーション" >}}

ページを更新すると、本の評価が変化するはずです。
なぜなら、リクエストは `reviews` サービスの異なるバージョンに分散されているからです。

## 下一步 {#next-steps}

[次の部分](../secure-and-visualize/)に進み、アプリケーションをメッシュに追加し、
アプリケーション間の通信を保護および可視化する方法を学びます。
