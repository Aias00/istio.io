---
title: Kubernetes クラスタのセットアップ
overview: チュートリアル用の Kubernetes クラスタを準備します。
weight: 2
owner: istio/wg-docs-maintainers
test: no
---

{{< boilerplate work-in-progress >}}

このモジュールでは、Istio をインストールした Kubernetes クラスタをセットアップし、チュートリアル全体で使用する名前空間も準備します。

{{< warning >}}
ワークショップなどで講師がクラスタを用意している場合は、
[ローカルマシンのセットアップ](/ja/docs/examples/microservices-istio/setup-local-computer)に進んでください。
{{</ warning >}}

1. [Kubernetes クラスタ](https://kubernetes.io/ja/docs/tutorials/kubernetes-basics/)へのアクセス権があることを確認してください。
   [Google Kubernetes Engine](https://cloud.google.com/kubernetes-engine/docs/quickstart) や
   [IBM Cloud Kubernetes Service](https://cloud.ibm.com/docs/containers?topic=containers-getting-started) などが利用できます。

1. チュートリアルで使う名前空間名を環境変数に設定します。
   どんな名前でも構いません（例: `tutorial`）。

   {{< text bash >}}
   $ export NAMESPACE=tutorial
   {{< /text >}}

1. 名前空間を作成します：

   {{< text bash >}}
   $ kubectl create namespace $NAMESPACE
   {{< /text >}}

   {{< tip >}}
   管理者の場合は、ユーザーごとに個別の名前空間を割り当てることができます。このチュートリアルは複数ユーザーが複数の名前空間で同時に作業することをサポートしています。
   {{< /tip >}}

1. `demo` 設定ファイルを使って [Istio をインストール](/ja/docs/setup/) します。

1. この例では [Kiali](/ja/docs/ops/integrations/kiali/)
   と [Prometheus](/ja/docs/ops/integrations/prometheus/) のアドオンを利用するため、
   それらもインストールします。以下のコマンドで全アドオンをインストールできます：

   {{< text bash >}}
   $ kubectl apply -f @samples/addons@
   {{< /text >}}

   {{< tip >}}
   アドオンのインストールでエラーが出た場合は、もう一度コマンドを実行してください。
   タイミングの問題で解決する場合があります。
   {{< /tip >}}

1. これらの共通 Istio サービス用に Kubernetes Ingress リソースを作成します。
   この段階で各サービスの詳細を理解している必要はありません。

   - [Grafana](https://grafana.com/docs/guides/getting_started/)
   - [Jaeger](https://www.jaegertracing.io/docs/1.13/getting-started/)
   - [Prometheus](https://prometheus.io/docs/prometheus/latest/getting_started/)
   - [Kiali](https://kiali.io/docs/installation/quick-start/)

   `kubectl` コマンドで yaml を編集して各サービスの Ingress リソースを作成します：

   {{< text bash >}}
   $ kubectl apply -f - <<EOF
   apiVersion: networking.k8s.io/v1
   kind: Ingress
   metadata:
   name: istio-system
   namespace: istio-system
   annotations:
   kubernetes.io/ingress.class: istio
   spec:
   rules:

   - host: my-istio-dashboard.io
     http:
     paths:
     - path: /
       pathType: Prefix
       backend:
       service:
       name: grafana
       port:
       number: 3000
   - host: my-istio-tracing.io
     http:
     paths:
     - path: /
       pathType: Prefix
       backend:
       service:
       name: tracing
       port:
       number: 9411
   - host: my-istio-logs-database.io
     http:
     paths:
     - path: /
       pathType: Prefix
       backend:
       service:
       name: prometheus
       port:
       number: 9090
   - host: my-kiali.io
     http:
     paths: - path: /
     pathType: Prefix
     backend:
     service:
     name: kiali
     port:
     number: 20001
     EOF
     {{< /text >}}

1. `istio-system` 名前空間のリソースを読み取り可能にするロールを作成します。以降の手順でユーザー権限を制限するために必要です。

   {{< text bash >}}
   $ kubectl apply -f - <<EOF
   kind: Role
   apiVersion: rbac.authorization.k8s.io/v1
   metadata:
   name: istio-system-access
   namespace: istio-system
   rules:

   - apiGroups: ["", "extensions", "apps"]
     resources: ["*"]
     verbs: ["get", "list"]
     EOF
     {{< /text >}}

1. 各ユーザー用のサービスアカウントを作成します：

   {{< text bash >}}
   $ kubectl apply -f - <<EOF
   apiVersion: v1
   kind: ServiceAccount
   metadata:
   name: ${NAMESPACE}-user
   namespace: $NAMESPACE
   EOF
   {{< /text >}}

1. 各ユーザーの権限を制限します。チュートリアルでは、ユーザーは自分の名前空間でリソースを作成し、`istio-system` 名前空間のリソースを読み取るだけで十分です。自分のクラスタを使う場合でも、他の名前空間への影響を避けるためにこの方法が推奨されます。

   各ユーザーの名前空間にリード・ライト権限を与えるロールを作成し、
   そのロールと `istio-system` リソース読み取りロールをバインドします：

   {{< text bash >}}
   $ kubectl apply -f - <<EOF
   kind: Role
   apiVersion: rbac.authorization.k8s.io/v1
   metadata:
   name: ${NAMESPACE}-access
   namespace: $NAMESPACE
   rules:

   - apiGroups: ["", "extensions", "apps", "networking.k8s.io", "networking.istio.io", "authentication.istio.io",
     "rbac.istio.io", "config.istio.io", "security.istio.io"]
     resources: ["*"]
     verbs: ["*"]

   ***

   kind: RoleBinding
   apiVersion: rbac.authorization.k8s.io/v1
   metadata:
   name: ${NAMESPACE}-access
   namespace: $NAMESPACE
   subjects:

   - kind: ServiceAccount
     name: ${NAMESPACE}-user
     namespace: $NAMESPACE
     roleRef:
     apiGroup: rbac.authorization.k8s.io
     kind: Role
     name: ${NAMESPACE}-access

   ***

   kind: RoleBinding
   apiVersion: rbac.authorization.k8s.io/v1
   metadata:
   name: ${NAMESPACE}-istio-system-access
   namespace: istio-system
   subjects:

   - kind: ServiceAccount
     name: ${NAMESPACE}-user
     namespace: $NAMESPACE
     roleRef:
     apiGroup: rbac.authorization.k8s.io
     kind: Role
     name: istio-system-access
     EOF
     {{< /text >}}

1. 各ユーザーは自分専用の Kubernetes 設定ファイルが必要です。この設定ファイルにはクラスタ情報、サービスアカウント、証明書、ユーザーの名前空間が記載されています。`kubectl` コマンドはこの設定ファイルを使ってクラスタを操作します。

   各ユーザー用の Kubernetes 設定ファイルを作成します：

   {{< tip >}}
   このコマンドはクラスタ名が `tutorial-cluster` であることを前提としています。
   クラスタ名が異なる場合は、すべての参照を適宜置き換えてください。
   {{</ tip >}}

   {{< text bash >}}
   $ cat <<EOF > ./${NAMESPACE}-user-config.yaml
   apiVersion: v1
   kind: Config
   preferences: {}

   clusters:

   - cluster:
     certificate-authority-data: $(kubectl get secret $(kubectl get sa ${NAMESPACE}-user -n $NAMESPACE -o jsonpath={.secrets..name}) -n $NAMESPACE -o jsonpath='{.data.ca\.crt}')
        server: $(kubectl config view -o jsonpath="{.clusters[?(.name==\"$(kubectl config view -o jsonpath=\"{.contexts[?(.name==\\\"$(kubectl config current-context)\\\")].context.cluster}\")\")].cluster.server}")
     name: ${NAMESPACE}-cluster

   users:

   - name: ${NAMESPACE}-user
     user:
     as-user-extra: {}
     client-key-data: $(kubectl get secret $(kubectl get sa ${NAMESPACE}-user -n $NAMESPACE -o jsonpath={.secrets..name}) -n $NAMESPACE -o jsonpath='{.data.ca\.crt}')
     token: $(kubectl get secret $(kubectl get sa ${NAMESPACE}-user -n $NAMESPACE -o jsonpath={.secrets..name}) -n $NAMESPACE -o jsonpath={.data.token} | base64 --decode)

   contexts:

   - context:
     cluster: ${NAMESPACE}-cluster
     namespace: ${NAMESPACE}
     user: ${NAMESPACE}-user
     name: ${NAMESPACE}

   current-context: ${NAMESPACE}
   EOF
   {{< /text >}}

1. `${NAMESPACE}-user-config.yaml` 設定ファイル用に `KUBECONFIG` 環境変数を設定します：

   {{< text bash >}}
   $ export KUBECONFIG=$PWD/${NAMESPACE}-user-config.yaml
   {{< /text >}}

1. 設定ファイルが有効になっているか、現在の名前空間を表示して確認します：

   {{< text bash >}}
   $ kubectl config view -o jsonpath="{.contexts[?(@.name==\"$(kubectl config current-context)\")].context.namespace}"
   tutorial
   {{< /text >}}

   出力に名前空間名が表示されていれば OK です。

1. 自分でクラスタをセットアップした場合は、前述の `${NAMESPACE}-user-config.yaml` ファイルをローカルマシンにコピーしてください。`${NAMESPACE}` は前の手順で設定した名前空間名です（例: `tutorial-user-config.yaml`）。このファイルはチュートリアル中に再度使用します。

   講師の場合は、生成した設定ファイルを各受講者に配布してください。受講者はこの設定ファイルを自分のローカルマシンにコピーする必要があります。

おめでとうございます。チュートリアル用のクラスタセットアップが完了しました！

これで[ローカルマシンのセットアップ](/ja/docs/examples/microservices-istio/setup-local-computer)に進む準備ができました。
