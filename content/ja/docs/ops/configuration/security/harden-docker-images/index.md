---
title: Docker コンテナイメージの強化
description: 強化されたコンテナイメージを使って Istio の攻撃対象領域を減らす方法。
weight: 80
aliases:
  - /zh/help/ops/security/harden-docker-images
  - /zh/docs/ops/security/harden-docker-images
owner: istio/wg-security-maintainers
test: n/a
status: Beta
---

Istio の[デフォルトイメージ](https://hub.docker.com/r/istio/base)は `ubuntu` ベースで、いくつかの追加ツールが含まれています。
[Distroless イメージ](https://github.com/GoogleContainerTools/distroless)ベースの代替イメージも利用できます。

Distroless を使う場合、Distroless イメージには不要な実行ファイルやライブラリが含まれていません。

- 攻撃対象領域が減少し、脆弱性が最小限になります。
- イメージが小さくなり、起動も高速です。

公式 Distroless README の[なぜ Distroless イメージを使うのか？](https://github.com/GoogleContainerTools/distroless#why-should-i-use-distroless-images) セクションも参照してください。

## Distroless イメージのインストール {#install-distroless-images}

[インストール手順](/ja/docs/setup/install/istioctl/)に従って Istio をセットアップします。
**Distroless イメージ**を使うには `variant` オプションを追加します。

{{< text bash >}}
$ istioctl install --set values.global.variant=distroless
{{< /text >}}

注入されるプロキシイメージだけで Distroless を使いたい場合は、
[Proxy Config](/ja/docs/reference/config/networking/proxy-config/#ProxyImage) の `image.imageType` フィールドも利用できます。
上記の `variant` フラグはこのフィールドも自動で設定します。

## デバッグ {#debugging}

Distroless イメージにはすべてのデバッグツール（シェルも含む）がありません。
これはセキュリティ上は有利ですが、`kubectl exec` でプロキシコンテナを一時的にデバッグすることができなくなります。

幸い、[エフェメラルコンテナ（臨時コンテナ）](https://kubernetes.io/ja/docs/concepts/workloads/pods/ephemeral-containers/)が役立ちます。
`kubectl debug` で Pod に臨時コンテナをアタッチできます。
追加ツール入りのイメージを使えば、従来通りデバッグが可能です：

{{< text shell >}}
$ kubectl debug --image istio/base --target istio-proxy -it app-65c6749c9d-t549t
Defaulting debug container name to debugger-cdftc.
If you don't see a command prompt, try pressing enter.
root@app-65c6749c9d-t549t:/# curl example.com
{{< /text >}}

このコマンドは `istio/base` を使って新しい臨時コンテナをデプロイします。
これは Distroless Istio イメージのベースイメージと同じで、Istio のデバッグに役立つさまざまなツールが含まれています。
ただし、どんなイメージでも利用可能です。
このコンテナは Sidecar プロキシ（`--target istio-proxy`）のプロセス名前空間と Pod のネットワーク名前空間にもアタッチされます。
