---
title: Sidecar 自動インジェクションの問題
description: Istio が Kubernetes Webhook を使って Sidecar を自動インジェクションする際のよくある問題の解決方法。
force_inline_toc: true
weight: 40
aliases:
  - /zh/docs/ops/troubleshooting/injection
owner: istio/wg-user-experience-maintainers
test: n/a
---

## インジェクション結果が期待と異なる {#the-result-of-sidecar-injection-was-not-what-i-expected}

期待しない Sidecar のインジェクションや、期待したのにインジェクションされない場合が含まれます。

1. Pod が `kube-system` または `kube-public` 名前空間にないことを確認してください。
   これらの名前空間の Pod には Sidecar 自動インジェクションは適用されません。

1. Pod の定義で `hostNetwork: true` になっていないことを確認してください。
   `hostNetwork: true` の Pod には Sidecar 自動インジェクションは適用されません。

   Sidecar モデルは iptables が Pod 内のすべてのトラフィックを Envoy にリダイレクトすることを前提としていますが、
   `hostNetwork: true` の Pod ではこの前提が成り立たず、ホストレベルのルーティング失敗を引き起こします。

1. webhook の `namespaceSelector` を確認し、対象の名前空間が webhook の範囲内かどうかを調べてください。

   範囲内の `namespaceSelector` 例：

   {{< text bash yaml >}}
   $ kubectl get mutatingwebhookconfiguration istio-sidecar-injector -o yaml | grep "namespaceSelector:" -A5
   namespaceSelector:
   matchLabels:
   istio-injection: enabled
   rules:

   - apiGroups: - ""
     {{< /text >}}

   `istio-injection=enabled` ラベルが付いた名前空間で Pod を作成すると、インジェクション webhook が呼び出されます。

   {{< text bash >}}
   $ kubectl get namespace -L istio-injection
   NAME STATUS AGE ISTIO-INJECTION
   default Active 18d enabled
   istio-system Active 3d
   kube-public Active 18d
   kube-system Active 18d
   {{< /text >}}

   インジェクション範囲外の `namespaceSelector` 例：

   {{< text bash >}}
   $ kubectl get mutatingwebhookconfiguration istio-sidecar-injector -o yaml | grep "namespaceSelector:" -A5
   namespaceSelector:
   matchExpressions: - key: istio-injection
   operator: NotIn
   values: - disabled
   rules:

   - apiGroups: - ""
     {{< /text >}}

   `istio-injection=disabled` ラベルが付いていない名前空間で Pod を作成すると、インジェクション webhook が呼び出されます。

   {{< text bash >}}
   $ kubectl get namespace -L istio-injection
   NAME STATUS AGE ISTIO-INJECTION
   default Active 18d
   istio-system Active 3d disabled
   kube-public Active 18d disabled
   kube-system Active 18d disabled
   {{< /text >}}

   アプリケーション Pod の名前空間が正しく（再）ラベル付けされているか確認してください。例：

   {{< text bash >}}
   $ kubectl label namespace istio-system istio-injection=disabled --overwrite
   {{< /text >}}

   （自動インジェクション webhook が必要なすべての名前空間で上記を繰り返してください）

   {{< text bash >}}
   $ kubectl label namespace default istio-injection=enabled --overwrite
   {{< /text >}}

1. デフォルトポリシーの確認

   `istio-sidecar-injector configmap` でデフォルトのインジェクションポリシーを確認します。

   {{< text bash yaml >}}
   $ kubectl -n istio-system get configmap istio-sidecar-injector -o jsonpath='{.data.config}' | grep policy:
   policy: enabled
   {{< /text >}}

   ポリシーの値は `disabled` または `enabled` です。webhook の `namespaceSelector` が対象名前空間と一致する場合のみ、デフォルトポリシーが有効です。不明な値の場合、インジェクションは完全に無効化されます。

1. 各 Pod のラベルを確認

   pod template spec metadata のラベル `sidecar.istio.io/inject` でデフォルトポリシーを上書きできます。この場合、Deployment の metadata は無視されます。ラベル値が `true` なら Sidecar 強制インジェクション、`false` なら強制非インジェクションです。

   以下のラベルはデフォルトポリシーを上書きし、Sidecar を強制インジェクションします：

   {{< text bash yaml >}}
   $ kubectl get deployment curl -o yaml | grep "sidecar.istio.io/inject:" -B4
   template:
   metadata:
   labels:
   app: curl
   sidecar.istio.io/inject: "true"
   {{< /text >}}

## Pod が作成できない {#pods-cannot-be-created-at-all}

失敗した Pod の Deployment で `kubectl describe -n namespace deployment name` を実行してください。
イベントにインジェクション webhook 失敗の理由が表示されることが多いです。

### x509 証明書関連のエラー {#x509-certificate-related-errors}

{{< text plain >}}
Warning FailedCreate 3m (x17 over 8m) replicaset-controller Error creating: Internal error occurred: \
 failed calling admission webhook "sidecar-injector.istio.io": Post https://istio-sidecar-injector.istio-system.svc:443/inject: \
 x509: certificate signed by unknown authority (possibly because of "crypto/rsa: verification error" while trying \
 to verify candidate authority certificate "Kubernetes.cluster.local")
{{< /text >}}

`x509: certificate signed by unknown authority` エラーは、webhook 設定の空の `caBundle` が原因で発生することが多いです。

`mutatingwebhookconfiguration` 設定の `caBundle` が `istio-sidecar-injector` Pod にインストールされているルート証明書と一致しているか確認してください。

{{< text bash >}}
$ kubectl get mutatingwebhookconfiguration istio-sidecar-injector -o yaml -o jsonpath='{.webhooks[0].clientConfig.caBundle}' | md5sum
4b95d2ba22ce8971c7c92084da31faf0 -
$ kubectl -n istio-system get configmap istio-ca-root-cert -o jsonpath='{.data.root-cert\.pem}' | base64 -w 0 | md5sum
4b95d2ba22ce8971c7c92084da31faf0 -
{{< /text >}}

CA 証明書が一致しない場合は、sidecar-injector Pod を再起動してください。

{{< text bash >}}
$ kubectl -n istio-system patch deployment istio-sidecar-injector \
 -p "{\"spec\":{\"template\":{\"metadata\":{\"labels\":{\"date\":\"`date +'%s'`\"}}}}}"
deployment.extensions "istio-sidecar-injector" patched
{{< /text >}}

### Deployment ステータスエラー {#errors-in-deployment-status}

Pod で自動 Sidecar インジェクションを有効にしている場合、何らかの理由でインジェクションに失敗すると、Pod の作成も失敗します。この場合、Pod のデプロイメントステータスを確認してエラーを特定できます。
これらのエラーは、デプロイメントが属する名前空間のイベントにも表示されます。

たとえば、Pod をデプロイしようとしたときに `istiod` コントロールプレーン Pod が稼働していない場合、イベントには次のようなエラーが表示されます：

{{< text bash >}}
$ kubectl get events -n curl
...
23m Normal SuccessfulCreate replicaset/curl-9454cc476 Created pod: curl-9454cc476-khp45
22m Warning FailedCreate replicaset/curl-9454cc476 Error creating: Internal error occurred: failed calling webhook "namespace.sidecar-injector.istio.io": failed to call webhook: Post "https://istiod.istio-system.svc:443/inject?timeout=10s": dial tcp 10.96.44.51:443: connect: connection refused
{{< /text >}}

{{< text bash >}}
$ kubectl -n istio-system get pod -lapp=istiod
NAME READY STATUS RESTARTS AGE
istiod-7d46d8d9db-jz2mh 1/1 Running 0 2d
{{< /text >}}

{{< text bash >}}
$ kubectl -n istio-system get endpoints istiod
NAME ENDPOINTS AGE
istiod 10.244.2.8:15012,10.244.2.8:15010,10.244.2.8:15017 + 1 more... 3h18m
{{< /text >}}

istiod Pod や endpoint がまだ準備できていない場合は、Pod のログやステータスを確認して Webhook Pod が起動できない理由を調べてください。

{{< text bash >}}
$ for pod in $(kubectl -n istio-system get pod -lapp=istiod -o jsonpath='{.items[*].metadata.name}'); do \
 kubectl -n istio-system logs ${pod} \
done

$ for pod in $(kubectl -n istio-system get pod -l app=istiod -o name); do \
kubectl -n istio-system describe ${pod}; \
done
$
{{< /text >}}

## Kubernetes API server にプロキシ設定がある場合、Sidecar の自動インジェクションは動作しない {#automatic-sidecar-injection-fails-if-the-Kubernetes-API-server-has-proxy-settings}

Kubernetes API server に以下のようなプロキシ設定がある場合：

{{< text yaml >}}
env:

- name: http_proxy
  value: http://proxy-wsa.esl.foo.com:80
- name: https_proxy
  value: http://proxy-wsa.esl.foo.com:80
- name: no_proxy
  value: 127.0.0.1,localhost,dockerhub.foo.com,devhub-docker.foo.com,10.84.100.125,10.84.100.126,10.84.100.127
  {{< /text >}}

このような設定では、Sidecar の自動インジェクションは失敗します。関連するエラーは `kube-apiserver` のログで確認できます：

{{< text plain >}}
W0227 21:51:03.156818 1 admission.go:257] Failed calling webhook, failing open sidecar-injector.istio.io: failed calling admission webhook "sidecar-injector.istio.io": Post https://istio-sidecar-injector.istio-system.svc:443/inject: Service Unavailable
{{< /text >}}

`*_proxy` 関連の環境変数設定に応じて、Pod や service CIDR がプロキシされていないことを確認してください。`kube-apiserver` の実行ログでリクエストがプロキシ経由になっていないか確認します。

1 つの解決策は `kube-apiserver` の設定からプロキシ設定を削除すること、もう 1 つは `istio-sidecar-injector.istio-system.svc` または `.svc` を `no_proxy` の `value` に追加することです。いずれの場合も `kube-apiserver` の再起動が必要です。

Kubernetes の関連 [issue](https://github.com/kubernetes/kubeadm/issues/666) は [PR #58698](https://github.com/kubernetes/kubernetes/pull/58698#discussion_r163879443) で解決されています。

## Pod 内での `tcpdump` 利用の制限 {#limitations-for-using-Tcpdump-in-pods}

`tcpdump` は Sidecar では動作しません（root 権限で動作しないため）。ただし、同じ Pod 内の他のコンテナはネットワーク名前空間を共有しているため、すべてのパケットを観測できます。
`iptables` も Pod レベルの設定を確認できます。

Envoy とアプリケーション間の通信は 127.0.0.1 経由で行われ、暗号化されていません。

## クラスタが自動縮小されない {#cluster-is-not-scaled-down-automatically}

Sidecar コンテナがローカルストレージボリュームをマウントしているため、ノード自動スケーラーはインジェクションされた Pod をノードから退避できません。これは[既知の問題](https://github.com/istio/istio/issues/19395)です。
回避策として、Pod にアノテーション `"cluster-autoscaler.kubernetes.io/safe-to-evict": "true"` を追加してください。

## istio-proxy が準備できていない場合、Pod やコンテナでネットワーク問題が発生する {#pod-or-containers-start-with-network-issues-if-istio-proxy-is-not-ready}

多くのアプリケーションは起動時にネットワーク接続が必要なコマンドやチェックを実行します。このとき istio-proxy Sidecar コンテナがまだ準備できていないと、アプリケーションコンテナがハングしたり再起動したりすることがあります。

これを防ぐには、`holdApplicationUntilProxyStarts` を `true` に設定してください。
これにより Sidecar インジェクターは Pod のコンテナリストの他のコンテナを起動する前に Sidecar を注入し、プロキシが準備できるまで他のコンテナの起動をブロックします。

これはグローバル設定オプションとして追加できます：

{{< text yaml >}}
values.global.proxy.holdApplicationUntilProxyStarts: true
{{< /text >}}

または Pod の annotation で指定できます：

{{< text yaml >}}
proxy.istio.io/config: '{ "holdApplicationUntilProxyStarts": true }'
{{< /text >}}
