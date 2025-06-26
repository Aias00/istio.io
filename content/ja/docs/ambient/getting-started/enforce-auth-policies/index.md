---
title: 認証ポリシーの実行
description: Ambient メッシュで四層と七層の認証ポリシーを実行します。
weight: 4
owner: istio/wg-networking-maintainers
test: yes
---

アプリケーションを Ambient メッシュに追加したら、四層認証ポリシーを使用してアプリケーションへのアクセスを保護できます。

この機能では、グリッド内のすべてのワークロードによって自動的に発行されたクライアントワークロードの ID に基づいて、サービスへのアクセスを制御できます。

## 四層認証ポリシーを実行 {#enforce-layer-4-authorization-policy}

[認証ポリシー](/zh/docs/reference/config/security/authorization-policy/)を作成して、どのサービスが `productpage` サービスと通信できるかを制限します。
このポリシーは、`app: productpage` ラベルが付けられた Pod に適用され、
サービスアカウント `cluster.local/ns/default/sa/bookinfo-gateway-istio` からの呼び出しだけを許可します。
これは、前のステップでデプロイした Bookinfo ゲートウェイで使用されるサービスアカウントです。

{{< text syntax=bash snip_id=deploy_l4_policy >}}
$ kubectl apply -f - <<EOF
apiVersion: security.istio.io/v1
kind: AuthorizationPolicy
metadata:
  name: productpage-ztunnel
  namespace: default
spec:
  selector:
    matchLabels:
      app: productpage
  action: ALLOW
  rules:
  - from:
    - source:
        principals:
        - cluster.local/ns/default/sa/bookinfo-gateway-istio
EOF
{{< /text >}}

ブラウザで Bookinfo アプリケーション（`http://localhost:8080/productpage`）を開くと、
前と同様に、製品ページが表示されます。しかし、異なるサービスアカウントから `productpage` サービスにアクセスしようとすると、エラーが表示されます。

クラスター内の異なるクライアントから Bookinfo アプリケーションにアクセスしてみましょう：

{{< text syntax=bash snip_id=deploy_curl >}}
$ kubectl apply -f @samples/curl/curl.yaml@
{{< /text >}}

`curl` Pod は異なるサービスアカウントを使用しているため、`productpage` サービスにアクセスできません：

{{< text bash >}}
$ kubectl exec deploy/curl -- curl -s "http://productpage:9080/productpage"
command terminated with exit code 56
{{< /text >}}

## 七層認証ポリシーを実行 {#enforce-layer-7-authorization-policy}

七層ポリシーを実行するには、まず、{{< gloss "waypoint" >}}waypoint プロキシ{{< /gloss >}}を名前空間にデプロイする必要があります。
このプロキシは、名前空間に入るすべての七層トラフィックを処理します。

{{< text syntax=bash snip_id=deploy_waypoint >}}
$ istioctl waypoint apply --enroll-namespace --wait
✅ waypoint default/waypoint applied
✅ waypoint default/waypoint is ready!
✅ namespace default labeled with "istio.io/use-waypoint: waypoint"
{{< /text >}}

waypoint プロキシを確認し、`Programmed=True` の状態であることを確認できます：

{{< text bash >}}
$ kubectl get gtw waypoint
NAME       CLASS            ADDRESS       PROGRAMMED   AGE
waypoint   istio-waypoint   10.96.58.95   True         42s
{{< /text >}}

[L7 認証ポリシー](/zh/docs/ambient/usage/l7-features/)を追加して、
`curl` サービスが `productpage` サービスに `GET` リクエストを送信できるようにし、
他の操作はできないようにします：

{{< text syntax=bash snip_id=deploy_l7_policy >}}
$ kubectl apply -f - <<EOF
apiVersion: security.istio.io/v1
kind: AuthorizationPolicy
metadata:
  name: productpage-waypoint
  namespace: default
spec:
  targetRefs:
  - kind: Service
    group: ""
    name: productpage
  action: ALLOW
  rules:
  - from:
    - source:
        principals:
        - cluster.local/ns/default/sa/curl
    to:
    - operation:
        methods: ["GET"]
EOF
{{< /text >}}

`targetRefs` フィールドは、waypoint プロキシ認証ポリシーの対象サービスを指定するために使用されます。
ルール部分は以前と同様ですが、今回は `to` 部分を追加して許可される操作を指定します。

前のステップで、L4 ポリシーは ztunnel がゲートウェイからの接続のみを許可するように指示しました。
今回は、waypoint からの接続も許可するように更新する必要があります。

{{< text syntax=bash snip_id=update_l4_policy >}}
$ kubectl apply -f - <<EOF
apiVersion: security.istio.io/v1
kind: AuthorizationPolicy
metadata:
  name: productpage-ztunnel
  namespace: default
spec:
  selector:
    matchLabels:
      app: productpage
  action: ALLOW
  rules:
  - from:
    - source:
        principals:
        - cluster.local/ns/default/sa/bookinfo-gateway-istio
        - cluster.local/ns/default/sa/waypoint
EOF
{{< /text >}}

{{< tip >}}
より多くの Istio 機能を有効にする方法については、[七層機能ユーザーガイド](/zh/docs/ambient/usage/l7-features/)を参照してください。
{{< /tip >}}

更新された認証ポリシーが新しい waypoint プロキシによって実行されていることを確認します：

{{< text bash >}}
$ # GET 操作を使用していないため、この操作は失敗し、RBAC エラーが表示されます
$ kubectl exec deploy/curl -- curl -s "http://productpage:9080/productpage" -X DELETE
RBAC: access denied
{{< /text >}}

{{< text bash >}}
$ # reviews-v1 サービスの ID が許可されていないため、この操作は失敗し、RBAC エラーが表示されます
$ kubectl exec deploy/reviews-v1 -- curl -s http://productpage:9080/productpage
RBAC: access denied
{{< /text >}}

{{< text bash >}}
$ # これは有効です。なぜなら、curl Pod からの GET リクエストを明示的に許可したからです
$ kubectl exec deploy/curl -- curl -s http://productpage:9080/productpage | grep -o "<title>.*</title>"
<title>Simple Bookstore App</title>
{{< /text >}}

## 下一步 {#next-steps}

waypoint プロキシを使用すると、今や名前空間で七層ポリシーを実行できます。認証ポリシーに加えて、
[waypoint プロキシを使用してサービス間でトラフィックを分割することもできます](../manage-traffic/)。
これは、キャニスター部署や A/B テストを行う際に非常に便利です。
