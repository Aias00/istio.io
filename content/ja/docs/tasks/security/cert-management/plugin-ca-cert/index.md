---
title: CA 証明書の挿入
description: システム管理者がルート証明書、署名証明書、秘密鍵を使って Istio の CA を設定する方法を紹介します。
weight: 80
keywords: [security, certificates]
aliases:
  - /zh/docs/tasks/security/plugin-ca-cert/
owner: istio/wg-security-maintainers
test: yes
---

このタスクでは、システム管理者がルート証明書、署名証明書、秘密鍵を使って Istio 証明書認証局（CA）を設定する方法を紹介します。

デフォルトでは、Istio CA は自己署名のルート証明書と秘密鍵を生成し、それらを使ってワークロード証明書に署名します。
ルート CA の秘密鍵を保護するため、セキュアなマシン上でオフラインでルート CA を運用し、ルート CA から各クラスタで稼働する Istio CA に中間証明書を発行することを推奨します。Istio CA は管理者が指定した証明書と秘密鍵を使ってワークロード証明書に署名し、管理者が指定したルート証明書を信頼のルートとしてワークロードに配布できます。

下図は、2 つのクラスタを含むメッシュで推奨される CA 階層構造を示しています。

{{< image width="50%"
    link="ca-hierarchy.svg"
    caption="CA Hierarchy"
    >}}

このタスクでは、Istio CA の証明書と秘密鍵を生成し、挿入する方法を紹介します。これらの手順は繰り返し実行でき、各クラスタで稼働する Istio CA に証明書と秘密鍵を提供できます。

## クラスタに証明書と秘密鍵を挿入する{#plug-in-certificates-and-key-into-the-cluster}

{{< warning >}}
以下はデモ用です。プロダクション環境のクラスタ設定には [Hashicorp Vault](https://www.hashicorp.com/products/vault) などのプロダクション CA の利用を強く推奨します。強固なセキュリティ対策が施されたオフラインマシンでルート CA を管理するのが良いプラクティスです。
{{< /warning >}}

{{< warning >}}
[Go 1.18 ではデフォルトで](https://github.com/golang/go/issues/41682) SHA-1 署名のサポートが無効化されています。macOS で証明書を生成する場合は OpenSSL を使用してください。詳細は [GitHub issue 38049](https://github.com/istio/istio/issues/38049) を参照してください。
{{< /warning >}}

1. Istio インストールパッケージのトップディレクトリで、証明書と秘密鍵を保存するディレクトリを作成します：

   {{< text bash >}}
   $ mkdir -p certs
   $ pushd certs
   {{< /text >}}

1. ルート証明書と秘密鍵を生成します：

   {{< text bash >}}
   $ make -f ../tools/certs/Makefile.selfsigned.mk root-ca
   {{< /text >}}

   以下のファイルが生成されます：

   - `root-cert.pem`：生成されたルート証明書
   - `root-key.pem`：生成されたルート秘密鍵
   - `root-ca.conf`：ルート証明書生成用の `openssl` 設定
   - `root-cert.csr`：ルート証明書用に生成された CSR

1. 各クラスタ用に Istio CA の中間証明書と秘密鍵を生成します。
   例としてクラスタ `cluster1` の場合：

   {{< text bash >}}
   $ make -f ../tools/certs/Makefile.selfsigned.mk cluster1-cacerts
   {{< /text >}}

   上記コマンドを実行すると、`cluster1` ディレクトリに以下のファイルが生成されます：

   - `ca-cert.pem`：生成された中間証明書
   - `ca-key.pem`：生成された中間秘密鍵
   - `cert-chain.pem`：istiod が使用する証明書チェーン
   - `root-cert.pem`：ルート証明書

   `cluster1` の部分は任意の文字列に置き換え可能です。例えば `cluster2-cacerts` を指定すれば `cluster2` ディレクトリに証明書と秘密鍵が作成されます。

   オフラインマシンで作業した場合は、生成したディレクトリをクラスタにアクセスできるマシンにコピーしてください。

1. 各クラスタで、`ca-cert.pem`、`ca-key.pem`、`root-cert.pem`、`cert-chain.pem` を含む Secret `cacerts` を作成します。例：`cluster1` クラスタの場合：

   {{< text bash >}}
   $ kubectl create namespace istio-system
   $ kubectl create secret generic cacerts -n istio-system \
    --from-file=cluster1/ca-cert.pem \
    --from-file=cluster1/ca-key.pem \
    --from-file=cluster1/root-cert.pem \
    --from-file=cluster1/cert-chain.pem
   {{< /text >}}

1. Istio インストールのトップディレクトリに戻ります：

   {{< text bash >}}
   $ popd
   {{< /text >}}

## Istio をデプロイする{#deploy-Istio}

1. `demo` プロファイルで Istio をデプロイします。

   Istio の CA は Secret から証明書と秘密鍵を読み込みます。

   {{< text bash >}}
   $ istioctl install --set profile=demo
   {{< /text >}}

## サンプルサービスをデプロイする{#deploying-example-services}

1. `httpbin` と `curl` サンプルサービスをデプロイします。

   {{< text bash >}}
   $ kubectl create ns foo
   $ kubectl apply -f <(istioctl kube-inject -f samples/httpbin/httpbin.yaml) -n foo
   $ kubectl apply -f <(istioctl kube-inject -f samples/curl/curl.yaml) -n foo
   {{< /text >}}

1. `foo` 名前空間のワークロードに対し、相互 TLS トラフィックのみを許可するポリシーをデプロイします。

   {{< text bash >}}
   $ kubectl apply -n foo -f - <<EOF
   apiVersion: security.istio.io/v1
   kind: PeerAuthentication
   metadata:
   name: "default"
   spec:
   mtls:
   mode: STRICT
   EOF
   {{< /text >}}

## 証明書の検証{#verifying-the-certificates}

このセクションでは、ワークロード証明書が挿入した CA 証明書で署名されているかを検証します。検証には `openssl` が必要です。

1. `httpbin` の証明書チェーンを取得する前に、mTLS ポリシーが有効になるまで 20 秒待ちます。この例の CA 証明書は自己署名なので、openssl コマンドの出力に `verify error:num=19:self signed certificate in certificate chain` が含まれるのは想定通りです。

   {{< text bash >}}
   $ sleep 20; kubectl exec "$(kubectl get pod -l app=curl -n foo -o jsonpath={.items..metadata.name})" -c istio-proxy -n foo -- openssl s_client -showcerts -connect httpbin.foo:8000 > httpbin-proxy-cert.txt
   {{< /text >}}

1. 証明書チェーンから証明書を抽出します。

   {{< text bash >}}
   $ sed -n '/-----BEGIN CERTIFICATE-----/{:start /-----END CERTIFICATE-----/!{N;b start};/.\*/p}' httpbin-proxy-cert.txt > certs.pem
   $ awk 'BEGIN {counter=0;} /BEGIN CERT/{counter++} { print > "proxy-cert-" counter ".pem"}' < certs.pem
   {{< /text >}}

1. ルート証明書が管理者指定の証明書と一致するか確認します：

   {{< text bash >}}
   $ openssl x509 -in certs/cluster1/root-cert.pem -text -noout > /tmp/root-cert.crt.txt
   $ openssl x509 -in ./proxy-cert-3.pem -text -noout > /tmp/pod-root-cert.crt.txt
   $ diff -s /tmp/root-cert.crt.txt /tmp/pod-root-cert.crt.txt
   Files /tmp/root-cert.crt.txt and /tmp/pod-root-cert.crt.txt are identical
   {{< /text >}}

1. CA 証明書が管理者指定の証明書と一致するか確認します：

   {{< text bash >}}
   $ openssl x509 -in certs/cluster1/ca-cert.pem -text -noout > /tmp/ca-cert.crt.txt
   $ openssl x509 -in ./proxy-cert-2.pem -text -noout > /tmp/pod-cert-chain-ca.crt.txt
   $ diff -s /tmp/ca-cert.crt.txt /tmp/pod-cert-chain-ca.crt.txt
   Files /tmp/ca-cert.crt.txt and /tmp/pod-cert-chain-ca.crt.txt are identical
   {{< /text >}}

1. ルート証明書からワークロード証明書までの証明書チェーンを検証します：

   {{< text bash >}}
   $ openssl verify -CAfile <(cat certs/cluster1/ca-cert.pem certs/cluster1/root-cert.pem) ./proxy-cert-1.pem
   ./proxy-cert-1.pem: OK
   {{< /text >}}

## クリーンアップ{#cleanup}

- ローカルディスクから証明書、秘密鍵、中間ファイルを削除します：

  {{< text bash >}}
  $ rm -rf certs
  {{< /text >}}

- Secret `cacerts` を削除します：

  {{< text bash >}}
  $ kubectl delete secret cacerts -n istio-system
  {{< /text >}}

- `foo` 名前空間から認証ポリシーを削除します：

  {{< text bash >}}
  $ kubectl delete peerauthentication -n foo default
  {{< /text >}}

- サンプルアプリ `curl` と `httpbin` を削除します：

  {{< text bash >}}
  $ kubectl delete -f samples/curl/curl.yaml -n foo
  $ kubectl delete -f samples/httpbin/httpbin.yaml -n foo
  {{< /text >}}

- クラスタから Istio をアンインストールします：

  {{< text bash >}}
  $ istioctl uninstall --purge -y
  {{< /text >}}

- クラスタから `foo` および `istio-system` 名前空間を削除します：

  {{< text bash >}}
  $ kubectl delete ns foo istio-system
  {{< /text >}}
