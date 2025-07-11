---
title: istioctl check-inject で Istio Sidecar のインジェクションを検証する
description: istioctl check-inject を使って、Deployment で Istio Sidecar インジェクションが正しく有効になっているか確認する方法を解説します。
weight: 45
keywords: [istioctl, injection, kubernetes]
owner: istio/wg-user-experience-maintainers
test: no
---

`istioctl experimental check-inject` は、特定の Webhook が Pod に対して Istio Sidecar インジェクションを実行するかどうかを検証できる診断ツールです。このツールは、Sidecar インジェクションの設定がアクティブなクラスタで正しく適用されているかを確認するのに役立ちます。

## クイックスタート {#quick-start}

特定の Pod でなぜ Istio Sidecar インジェクションが発生した／しなかった（または発生する／しない）のかを確認するには、次のコマンドを実行します：

{{< text syntax=bash >}}
$ istioctl experimental check-inject -n <namespace> <pod-name>
{{< /text >}}

Deployment の場合は、次のコマンドを実行します：

{{< text syntax=bash >}}
$ istioctl experimental check-inject -n <namespace> deploy/<deployment-name>
{{< /text >}}

また、ラベルペアで指定する場合：

{{< text syntax=bash >}}
$ istioctl experimental check-inject -n <namespace> -l <label-key>=<label-value>
{{< /text >}}

例として、`hello` ネームスペースに `httpbin` という Deployment と、`app=httpbin` というラベルを持つ `httpbin-1234` という Pod がある場合、以下のコマンドは同等です：

{{< text syntax=bash >}}
$ istioctl experimental check-inject -n hello httpbin-1234
$ istioctl experimental check-inject -n hello deploy/httpbin
$ istioctl experimental check-inject -n hello -l app=httpbin
{{< /text >}}

出力例：

{{< text plain >}}
WEBHOOK REVISION INJECTED REASON
istio-revision-tag-default default ✔ Namespace label istio-injection=enabled matches
istio-sidecar-injector-1-18 1-18 ✘ No matching namespace labels (istio.io/rev=1-18) or pod labels (istio.io/rev=1-18)
{{< /text >}}

`INJECTED` フィールドが `✔` の場合、その行の Webhook がインジェクションを実行し、理由も表示されます。

`INJECTED` フィールドが `✘` の場合、その行の Webhook はインジェクションを実行せず、理由も表示されます。

Webhook がインジェクションを実行しない、またはエラーとなる主な理由：

1. **一致するネームスペースラベルや Pod ラベルがない**：ネームスペースや Pod に正しいラベルが設定されているか確認してください。

1. **特定のリビジョンに一致するネームスペースラベルや Pod ラベルがない**：必要な Istio リビジョンに一致するラベルを設定してください。

1. **インジェクションを防ぐ Pod ラベル**：ラベルを削除するか、適切な値に設定してください。

1. **インジェクションを防ぐネームスペースラベル**：ラベルを適切な値に変更してください。

1. **複数の Webhook が sidecar をインジェクションしている**：1 つの Webhook のみがインジェクションを行うようにし、ネームスペースや Pod に適切なラベルを設定して特定の Webhook をターゲットにしてください。
