---
title: トラストドメインの移行
description: 認可ポリシーを変更せずにトラストドメインを移行する方法を説明します。
weight: 60
keywords:
  [security, access-control, rbac, authorization, trust domain, migration]
owner: istio/wg-security-maintainers
test: yes
---

このタスクでは、認可ポリシーを変更せずにトラストドメインを移行する方法について説明します。

Istio 1.4 では、認可ポリシーの {{< gloss >}}trust domain migration{{</ gloss >}} をサポートする Alpha 機能が導入されました。
これは、Istio メッシュの {{< gloss >}}trust domain{{</ gloss >}} を変更する必要がある場合でも、認可ポリシーを手動で更新する必要がないことを意味します。Istio では、{{< gloss >}}workload{{</ gloss >}} が名前空間 `foo` でサービスアカウント `bar` で動作し、システムのトラストドメインが `my-td` の場合、そのワークロードの ID は `spiffe://my-td/ns/foo/sa/bar` となります。デフォルトでは、Istio メッシュのトラストドメインは `cluster.local` ですが、インストール時に別途指定しない限りこの値になります。

## 始める前に{#before-you-begin}

このタスクを始める前に、以下を完了してください：

1. [認可](/ja/docs/concepts/security/#authorization)ガイドを読んでください。

1. Istio をインストールし、トラストドメインをカスタマイズし、双方向 TLS を有効にします。

   {{< text bash >}}
   $ istioctl install --set profile=demo --set meshConfig.trustDomain=old-td
   {{< /text >}}

1. [httpbin]({{< github_tree >}}/samples/httpbin) サンプルを `default` 名前空間に、[curl]({{< github_tree >}}/samples/curl) サンプルを `default` と `curl-allow` 名前空間にデプロイします：

   {{< text bash >}}
   $ kubectl label namespace default istio-injection=enabled
   $ kubectl apply -f @samples/httpbin/httpbin.yaml@
   $ kubectl apply -f @samples/curl/curl.yaml@
   $ kubectl create namespace curl-allow
   $ kubectl label namespace curl-allow istio-injection=enabled
   $ kubectl apply -f @samples/curl/curl.yaml@ -n curl-allow
   {{< /text >}}

1. 次の認可ポリシーを適用し、`curl-allow` 名前空間の `curl` サービスからのリクエスト以外は `httpbin` へのすべてのリクエストを拒否します。

   {{< text bash >}}
   $ kubectl apply -f - <<EOF
   apiVersion: security.istio.io/v1
   kind: AuthorizationPolicy
   metadata:
   name: service-httpbin.default.svc.cluster.local
   namespace: default
   spec:
   rules:

   - from:
     - source:
       principals: - old-td/ns/curl-allow/sa/curl
       to:
     - operation:
       methods: - GET
       selector:
       matchLabels:
       app: httpbin

   ***

   EOF
   {{< /text >}}

   認可ポリシーが Sidecar に伝播するまで数十秒かかる場合があります。

1. 次のように `httpbin` へのリクエスト元を検証します：

   - `default` 名前空間の `curl` サービスからのリクエストは拒否されます。

     {{< text bash >}}
     $ kubectl exec "$(kubectl get pod -l app=curl -o jsonpath={.items..metadata.name})" -c curl -- curl http://httpbin.default:8000/ip -sS -o /dev/null -w "%{http_code}\n"
     403
     {{< /text >}}

   - `curl-allow` 名前空間の `curl` サービスからのリクエストは許可されます。

     {{< text bash >}}
     $ kubectl exec "$(kubectl -n curl-allow get pod -l app=curl -o jsonpath={.items..metadata.name})" -c curl -n curl-allow -- curl http://httpbin.default:8000/ip -sS -o /dev/null -w "%{http_code}\n"
     200
     {{< /text >}}

## トラストドメインをエイリアスなしで移行する{#migrate-trust-domain-without-trust-domain-aliases}

1. 新しいトラストドメインで Istio をインストールします。

   {{< text bash >}}
   $ istioctl install --set profile=demo --set meshConfig.trustDomain=new-td
   {{< /text >}}

1. istiod を再デプロイしてトラストドメインを変更します。

   {{< text bash >}}
   $ kubectl rollout restart deployment -n istio-system istiod
   {{< /text >}}

   Istio メッシュは新しいトラストドメイン `new-td` で動作するようになりました。

1. `httpbin` と `curl` アプリケーションを再デプロイし、新しい Istio コントロールプレーンから更新を取得します。

   {{< text bash >}}
   $ kubectl delete pod --all
   {{< /text >}}

   {{< text bash >}}
   $ kubectl delete pod --all -n curl-allow
   {{< /text >}}

1. `default` および `curl-allow` 名前空間の `curl` から `httpbin` へのアクセスがいずれも拒否されることを確認します。

   {{< text bash >}}
   $ kubectl exec "$(kubectl get pod -l app=curl -o jsonpath={.items..metadata.name})" -c curl -- curl http://httpbin.default:8000/ip -sS -o /dev/null -w "%{http_code}\n"
   403
   {{< /text >}}

   {{< text bash >}}
   $ kubectl exec "$(kubectl -n curl-allow get pod -l app=curl -o jsonpath={.items..metadata.name})" -c curl -n curl-allow -- curl http://httpbin.default:8000/ip -sS -o /dev/null -w "%{http_code}\n"
   403
   {{< /text >}}

   これは、`httpbin` へのすべてのリクエストを拒否する認可ポリシーを指定し、リクエスト元の ID が `old-td/ns/curl-allow/sa/curl` の場合のみ許可しているためです。
   新しいトラストドメイン `new-td` に移行すると、`curl` アプリケーションの ID は `new-td/ns/curl-allow/sa/curl` となり、`old-td/ns/curl-allow/sa/curl` とは異なります。そのため、以前は許可されていた `curl-allow` 名前空間の `curl` からのリクエストも拒否されます。Istio 1.4 より前は、この問題を解決する唯一の方法は認可ポリシーを手動で修正することでしたが、Istio 1.4 以降はより簡単な方法が導入されています。

## エイリアスを使ってトラストドメインを移行する{#migrate-trust-domain-with-trust-domain-aliases}

1. 新しいトラストドメインとトラストドメインエイリアスを指定して Istio をインストールします。

   {{< text bash >}}
   $ cat <<EOF > ./td-installation.yaml
   apiVersion: install.istio.io/v1alpha2
   kind: IstioControlPlane
   spec:
   meshConfig:
   trustDomain: new-td
   trustDomainAliases: - old-td
   EOF
   $ istioctl install --set profile=demo -f td-installation.yaml -y
   {{< /text >}}

1. 認可ポリシーを変更せずに `httpbin` へのリクエストを検証します：

   - `default` 名前空間の `curl` からのリクエストは拒否されます。

     {{< text bash >}}
     $ kubectl exec "$(kubectl get pod -l app=curl -o jsonpath={.items..metadata.name})" -c curl -- curl http://httpbin.default:8000/ip -sS -o /dev/null -w "%{http_code}\n"
     403
     {{< /text >}}

   - `curl-allow` 名前空間の `curl` からのリクエストは許可されます。

     {{< text bash >}}
     $ kubectl exec "$(kubectl -n curl-allow get pod -l app=curl -o jsonpath={.items..metadata.name})" -c curl -n curl-allow -- curl http://httpbin.default:8000/ip -sS -o /dev/null -w "%{http_code}\n"
     200
     {{< /text >}}

## ベストプラクティス{#best-practices}

Istio 1.4 以降、認可ポリシーを編集する際は、ポリシー内のトラストドメイン部分に `cluster.local` を使用することを推奨します。
例えば、`old-td/ns/curl-allow/sa/curl` ではなく、`cluster.local/ns/curl-allow/sa/curl` と記述します。
この場合、`cluster.local` は Istio メッシュのトラストドメイン（この例では `old-td`、後に `new-td`）およびそのエイリアスを指すポインタとなります。
認可ポリシーで `cluster.local` を使用することで、新しいトラストドメインに移行した際も、Istio はこの状況を検出し、新しいトラストドメインを旧ドメインと同等に扱います。

## クリーンアップ{#clean-up}

{{< text bash >}}
$ kubectl delete authorizationpolicy service-httpbin.default.svc.cluster.local
$ kubectl delete deploy httpbin; kubectl delete service httpbin; kubectl delete serviceaccount httpbin
$ kubectl delete deploy curl; kubectl delete service curl; kubectl delete serviceaccount curl
$ istioctl uninstall --purge -y
$ kubectl delete namespace curl-allow istio-system
$ rm ./td-installation.yaml
{{< /text >}}
