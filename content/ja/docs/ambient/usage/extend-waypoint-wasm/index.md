---
title: WebAssembly プラグインで waypoint を拡張する
description: Ambient モードでリモート WebAssembly モジュールを利用する方法を説明します。
weight: 55
keywords: [extensibility, Wasm, WebAssembly, Ambient]
owner: istio/wg-policies-and-telemetry-maintainers
test: yes
status: Alpha
---

{{< boilerplate alpha >}}

Istio は[WebAssembly（Wasm）による拡張機能](/ja/docs/concepts/wasm/)を提供しています。
Wasm 拡張性の主な利点の 1 つは、拡張プラグインをランタイムで動的にロードできることです。
本記事では、Istio で Wasm 機能を使って Ambient モードを拡張する方法の概要を説明します。
Ambient モードでは、各ネームスペースにデプロイされた waypoint プロキシに Wasm 設定を適用する必要があります。

## 始める前に {#before-you-begin}

1. [Ambient モード入門ガイド](/ja/docs/ambient/getting-started)の手順に従って Istio をセットアップします。
1. [Bookinfo サンプルアプリ](/ja/docs/ambient/getting-started/deploy-sample-app)をデプロイします。
1. [default ネームスペースを Ambient メッシュに追加](/ja/docs/ambient/getting-started/secure-and-visualize)します。
1. [curl]({{< github_tree >}}/samples/curl) サンプルアプリをデプロイし、リクエスト送信のテストソースとして利用します。

   {{< text syntax=bash >}}
   $ kubectl apply -f @samples/curl/curl.yaml@
   {{< /text >}}

## ゲートウェイで {#at-a-gateway}

Istio は Kubernetes Gateway API を利用し、サービスメッシュへのトラフィックを管理する集中エントリーポイントを提供します。
ここではゲートウェイレベルで WasmPlugin を設定し、ゲートウェイを通過するすべてのトラフィックが拡張認証ルールに従うようにします。

### ゲートウェイに WebAssembly プラグインを設定する {#configure-a-webassembly-plugin-for-a-gateway}

この例では、メッシュに HTTP
[ベーシック認証モジュール](https://github.com/istio-ecosystem/wasm-extensions/tree/master/extensions/basic_auth)を追加します。
Istio がリモートイメージレジストリからベーシック認証モジュールをプルしてロードするように設定します。
このモジュールは `/productpage` への呼び出し時に動作するように設定されます。
これらの手順は[WebAssembly モジュールの配布](/ja/docs/tasks/extensibility/wasm-module-distribution/)とほぼ同じですが、ラベルセレクタの代わりに `targetRefs` の利用が推奨されます。

リモート Wasm モジュールを使って WebAssembly フィルタを設定するには、`bookinfo-gateway` を指す `WasmPlugin` リソースを作成します：

{{< text syntax=bash snip_id=get_gateway >}}
$ kubectl get gateway
NAME CLASS ADDRESS PROGRAMMED AGE
bookinfo-gateway istio bookinfo-gateway-istio.default.svc.cluster.local True 42m
{{< /text >}}

{{< text syntax=bash snip_id=apply_wasmplugin_gateway >}}
$ kubectl apply -f - <<EOF
apiVersion: extensions.istio.io/v1alpha1
kind: WasmPlugin
metadata:
name: basic-auth-at-gateway
spec:
targetRefs: - kind: Gateway
group: gateway.networking.k8s.io
name: bookinfo-gateway # gateway name retrieved from previous step
url: oci://ghcr.io/istio-ecosystem/wasm-extensions/basic_auth:1.12.0
phase: AUTHN
pluginConfig:
basic_auth_rules: - prefix: "/productpage"
request_methods: - "GET" - "POST"
credentials: - "ok:test" - "YWRtaW4zOmFkbWluMw=="
EOF
{{< /text >}}

HTTP フィルタは認証フィルタとしてゲートウェイに注入されます。
Istio プロキシは WasmPlugin 設定を解釈し、OCI イメージレジストリからリモート Wasm モジュールをローカルファイルにダウンロードし、そのファイルを参照してゲートウェイに HTTP フィルタを注入します。

### ゲートウェイ経由でトラフィックを検証する {#verify-the-traffic-via-the-gateway}

1. 資格情報なしで `/productpage` をテスト：

   {{< text syntax=bash snip_id=test_gateway_productpage_without_credentials >}}
   $ kubectl exec deploy/curl -- curl -s -w "%{http_code}" -o /dev/null "http://bookinfo-gateway-istio.default.svc.cluster.local/productpage"
   401
   {{< /text >}}

1. WasmPlugin リソースで設定した資格情報を使って `/productpage` をテスト：

   {{< text syntax=bash snip_id=test_gateway_productpage_with_credentials >}}
   $ kubectl exec deploy/curl -- curl -s -o /dev/null -H "Authorization: Basic YWRtaW4zOmFkbWluMw==" -w "%{http_code}" "http://bookinfo-gateway-istio.default.svc.cluster.local/productpage"
   200
   {{< /text >}}

## waypoint でネームスペース内のすべてのサービスに適用する {#apply-wasm-configuration-at-waypoint-proxy}

waypoint プロキシは Istio の Ambient モードで重要な役割を果たします。サービスメッシュ内の通信の安全性と効率性を確保します。
以下では、waypoint に Wasm 設定を適用し、プロキシ機能を動的に拡張する方法を説明します。

### waypoint プロキシをデプロイする {#deploy-a-waypoint-proxy}

[waypoint デプロイ手順](/ja/docs/ambient/usage/waypoint/#deploy-a-waypoint-proxy)に従い、bookinfo ネームスペースに waypoint プロキシをデプロイします。

{{< text syntax=bash snip_id=create_waypoint >}}
$ istioctl waypoint apply --enroll-namespace --wait
{{< /text >}}

サービスへのトラフィックを検証します：

{{< text syntax=bash snip_id=verify_traffic >}}
$ kubectl exec deploy/curl -- curl -s -w "%{http_code}" -o /dev/null http://productpage:9080/productpage
200
{{< /text >}}

### waypoint に WebAssembly プラグインを設定する {#configure-a-webassembly-plugin-for-a-waypoint}

リモート Wasm モジュールを使って WebAssembly フィルタを設定するには、`waypoint` ゲートウェイを指す `WasmPlugin` リソースを作成します：

{{< text syntax=bash snip_id=get_gateway_waypoint >}}
$ kubectl get gateway
NAME CLASS ADDRESS PROGRAMMED AGE
bookinfo-gateway istio bookinfo-gateway-istio.default.svc.cluster.local True 23h
waypoint istio-waypoint 10.96.202.82 True 21h
{{< /text >}}

{{< text syntax=bash snip_id=apply_wasmplugin_waypoint_all >}}
$ kubectl apply -f - <<EOF
apiVersion: extensions.istio.io/v1alpha1
kind: WasmPlugin
metadata:
name: basic-auth-at-waypoint
spec:
targetRefs: - kind: Gateway
group: gateway.networking.k8s.io
name: waypoint # gateway name retrieved from previous step
url: oci://ghcr.io/istio-ecosystem/wasm-extensions/basic_auth:1.12.0
phase: AUTHN
pluginConfig:
basic_auth_rules: - prefix: "/productpage"
request_methods: - "GET" - "POST"
credentials: - "ok:test" - "YWRtaW4zOmFkbWluMw=="
EOF
{{< /text >}}

### 設定済みプラグインの確認 {#view-the-configured-plugin}

{{< text syntax=bash snip_id=get_wasmplugin >}}
$ kubectl get wasmplugin
NAME AGE
basic-auth-at-gateway 28m
basic-auth-at-waypoint 14m
{{< /text >}}

### waypoint プロキシ経由でトラフィックを検証する {#verify-the-traffic-via-waypoint-proxy}

1. 資格情報なしで内部 `/productpage` をテスト：

   {{< text syntax=bash snip_id=test_waypoint_productpage_without_credentials >}}
   $ kubectl exec deploy/curl -- curl -s -w "%{http_code}" -o /dev/null http://productpage:9080/productpage
   401
   {{< /text >}}

1. 資格情報ありで内部 `/productpage` をテスト：

   {{< text syntax=bash snip_id=test_waypoint_productpage_with_credentials >}}
   $ kubectl exec deploy/curl -- curl -s -w "%{http_code}" -o /dev/null -H "Authorization: Basic YWRtaW4zOmFkbWluMw==" http://productpage:9080/productpage
   200
   {{< /text >}}

## waypoint で特定サービスに適用する {#at-a-waypoint-for-a-specific-service}

特定サービスにリモート Wasm モジュールを使って WebAssembly フィルタを設定するには、対象サービスを指す WasmPlugin リソースを作成します。

`reviews` サービスを指す `WasmPlugin` を作成し、この拡張が `reviews` サービスのみに適用されるようにします。
この設定では、認証トークンとプレフィックスが `reviews` サービス専用にカスタマイズされており、そのサービスへのリクエストのみがこの認証メカニズムの影響を受けます。

{{< text syntax=bash snip_id=apply_wasmplugin_waypoint_service >}}
$ kubectl apply -f - <<EOF
apiVersion: extensions.istio.io/v1alpha1
kind: WasmPlugin
metadata:
name: basic-auth-for-service
spec:
targetRefs: - kind: Service
group: ""
name: reviews
url: oci://ghcr.io/istio-ecosystem/wasm-extensions/basic_auth:1.12.0
phase: AUTHN
pluginConfig:
basic_auth_rules: - prefix: "/reviews"
request_methods: - "GET" - "POST"
credentials: - "ok:test" - "MXQtaW4zOmFkbWluMw=="
EOF
{{< /text >}}

### サービス宛トラフィックの検証 {#verify-the-traffic-targeting-the-service}

1. 一般的な `waypoint` プロキシで設定した資格情報で内部 `/productpage` をテスト：

   {{< text syntax=bash snip_id=test_waypoint_service_productpage_with_credentials >}}
   $ kubectl exec deploy/curl -- curl -s -w "%{http_code}" -o /dev/null -H "Authorization: Basic YWRtaW4zOmFkbWluMw==" http://productpage:9080/productpage
   200
   {{< /text >}}

1. 特定の `reviews-svc-waypoint` プロキシで設定した資格情報で内部 `/reviews` をテスト：

   {{< text syntax=bash snip_id=test_waypoint_service_reviews_with_credentials >}}
   $ kubectl exec deploy/curl -- curl -s -w "%{http_code}" -o /dev/null -H "Authorization: Basic MXQtaW4zOmFkbWluMw==" http://reviews:9080/reviews/1
   200
   {{< /text >}}

1. 資格情報なしで内部 `/reviews` をテスト：

   {{< text syntax=bash snip_id=test_waypoint_service_reviews_without_credentials >}}
   $ kubectl exec deploy/curl -- curl -s -w "%{http_code}" -o /dev/null http://reviews:9080/reviews/1
   401
   {{< /text >}}

資格情報なしでコマンドを実行すると、内部 `/productpage` へのアクセスが 401 Unauthorized となり、正しい認証情報がない場合にリソースへのアクセスが失敗することが確認できます。

## クリーンアップ {#cleanup}

1. WasmPlugin 設定を削除します：

   {{< text syntax=bash snip_id=remove_wasmplugin >}}
   $ kubectl delete wasmplugin basic-auth-at-gateway basic-auth-at-waypoint basic-auth-for-service
   {{< /text >}}

1. [Ambient モードのアンインストールガイド](/ja/docs/ambient/getting-started/#uninstall)を参照し、Istio およびサンプルテストアプリを削除します。
