---
title: Istio ワークロードの最小 TLS バージョン設定
description: Istio ワークロードの最小 TLS バージョンを設定する方法を紹介します。
weight: 90
keywords: [安全, TLS]
aliases:
  - /zh/docs/tasks/security/workload-min-tls-version/
owner: istio/wg-security-maintainers
test: yes
---

このタスクでは、Istio ワークロードの最小 TLS バージョンを設定する方法を紹介します。
Istio ワークロードが現在サポートする最大の TLS バージョンは 1.3 です。

## Istio ワークロードの最小 TLS バージョンを設定する{#configuration-of-minimum-tls-version-for-Istio-workloads}

- `istioctl` を使って Istio をインストールし、最小 TLS バージョンを設定します。
  `istioctl install` コマンドで使用する Istio の `IstioOperator` カスタムリソースの YAML 設定内に、
  Istio ワークロードの最小 TLS バージョンを設定するフィールドがあります。
  その中の `minProtocolVersion` フィールドで、Istio ワークロード間の TLS 接続の最小バージョンを指定します。
  下記の例では、Istio ワークロードの最小 TLS バージョンを 1.3 に設定しています。

  {{< text bash >}}
  $ cat <<EOF > ./istio.yaml
  apiVersion: install.istio.io/v1alpha1
  kind: IstioOperator
  spec:
  meshConfig:
  meshMTLS:
  minProtocolVersion: TLSV1_3
  EOF
  $ istioctl install -f ./istio.yaml
  {{< /text >}}

## Istio ワークロードの TLS 設定を確認する{#check-the-tls-configuration-of-Istio-workloads}

Istio ワークロードの最小 TLS バージョンを設定した後、
最小バージョンの TLS が正しく設定され、期待通りに動作しているかを検証できます。

- 2 つのワークロード（`httpbin` と `curl`）をデプロイします。これらを同じ名前空間（例：`foo`）にデプロイし、
  どちらのワークロードも各サービスの前段で Envoy をトラフィックプロキシとして動作させます。

  {{< text bash >}}
  $ kubectl create ns foo
  $ kubectl apply -f <(istioctl kube-inject -f @samples/httpbin/httpbin.yaml@) -n foo
  $ kubectl apply -f <(istioctl kube-inject -f @samples/curl/curl.yaml@) -n foo
  {{< /text >}}

- 下記コマンドで `curl` から `httpbin` への通信が正常に行えることを確認します：

  {{< text bash >}}
  $ kubectl exec "$(kubectl get pod -l app=curl -n foo -o jsonpath={.items..metadata.name})" -c curl -n foo -- curl http://httpbin.foo:8000/ip -sS -o /dev/null -w "%{http_code}\n"
  200
  {{< /text >}}

{{< warning >}}
期待される出力が表示されない場合は、数秒後に再試行してください。
キャッシュや伝播の遅延が原因となる場合があります。
{{< /warning >}}

この例では、最小 TLS バージョンが 1.3 に設定されています。
以下のコマンドで TLS 1.3 プロトコルが許可されているか確認できます：

{{< text bash >}}
$ kubectl exec "$(kubectl get pod -l app=curl -n foo -o jsonpath={.items..metadata.name})" -c istio-proxy -n foo -- openssl s_client -alpn istio -tls1_3 -connect httpbin.foo:8000 | grep "TLSv1.3"
{{< /text >}}

出力には次のような内容が含まれるはずです：

{{< text plain >}}
TLSv1.3
{{< /text >}}

TLS 1.2 バージョンが許可されているかを確認するには、次のコマンドを実行します：

{{< text bash >}}
$ kubectl exec "$(kubectl get pod -l app=curl -n foo -o jsonpath={.items..metadata.name})" -c istio-proxy -n foo -- openssl s_client -alpn istio -tls1_2 -connect httpbin.foo:8000 | grep "Cipher is (NONE)"
{{< /text >}}

出力には次のような内容が含まれるはずです：

{{< text plain >}}
Cipher is (NONE)
{{< /text >}}

## クリーンアップ{#cleanup}

`foo` 名前空間からサンプルアプリ `curl` と `httpbin` を削除します：

{{< text bash >}}
$ kubectl delete -f samples/httpbin/httpbin.yaml -n foo
$ kubectl delete -f samples/curl/curl.yaml -n foo
{{< /text >}}

クラスタから Istio をアンインストールします：

{{< text bash >}}
$ istioctl uninstall --purge -y
{{< /text >}}

`foo` および `istio-system` 名前空間を削除します：

{{< text bash >}}
$ kubectl delete ns foo istio-system
{{< /text >}}
