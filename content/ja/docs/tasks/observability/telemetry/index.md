---
title: Telemetry API
description: このタスクでは Telemetry API の設定方法を紹介します。
weight: 0
keywords: [telemetry]
owner: istio/wg-policies-and-telemetry-maintainers
test: no
status: Stable
---

Istio は [Telemetry API](/zh/docs/reference/config/telemetry/) を提供しており、
柔軟に[メトリクス](/zh/docs/tasks/observability/metrics/)、
[アクセスログ](/zh/docs/tasks/observability/logs/)、[分散トレーシング](/zh/docs/tasks/observability/distributed-tracing/)を設定できます。

## API の使い方 {#using-api}

### スコープ、継承、オーバーライド {#scope-inheritance-and-overrides}

Istio の設定階層では、Telemetry API リソースは親リソースから設定を継承します：

1.  ルート設定ネームスペース（例：`istio-system`）
1.  ローカルネームスペース（ワークロード `selector` なしでネームスペース全体に作用するリソース）
1.  ワークロード（ワークロード `selector` 付きでネームスペースに作用するリソース）

`istio-system` などのルート設定ネームスペース内の Telemetry API リソースはメッシュ全体のデフォルト動作を提供します。
ルート設定ネームスペース内のワークロードセレクタ付きリソースは無視/拒否されます。
ルート設定ネームスペースで複数のメッシュ範囲 Telemetry API リソースを定義するのは無効です。

新しい `Telemetry` リソースを（ワークロードセレクタなしで）ターゲットネームスペースに適用することで、
メッシュ範囲設定に対してネームスペース単位のオーバーライドが可能です。ネームスペース設定で指定したフィールドは、
（ルート設定ネームスペースの）親設定のフィールドを完全に上書きします。

**ワークロードセレクタを使う**ことで、ターゲットネームスペースに新しい Telemetry リソースを適用し、
ワークロード単位のオーバーライドが可能です。

### ワークロード選択 {#workload-selection}

ネームスペース内の個々のワークロードは、[`selector`](/zh/docs/reference/config/type/workload-selector/#WorkloadSelector)
で選択できます。これにより、ラベルベースでワークロードを選択できます。

`selector` を使って 2 つの異なる `Telemetry` リソースが同じワークロードを選択するのは無効です。
また、`selector` を指定せずに 1 つのネームスペース内で 2 つの異なる `Telemetry` リソースを作成するのも無効です。

### プロバイダー選択 {#provider-selection}

Telemetry API では、利用する統合プロトコルやタイプを示す「プロバイダー」の概念を使います。
プロバイダーは [`MeshConfig`](/zh/docs/reference/config/istio.mesh.v1alpha1/#MeshConfig-ExtensionProvider)
で設定できます。

`MeshConfig` でのプロバイダー設定例は以下の通りです：

{{< text yaml >}}
data:
mesh: |-
extensionProviders: # 以下は 2 つのサンプル分散トレーシングプロバイダーの定義です。 - name: "localtrace"
zipkin:
service: "zipkin.istio-system.svc.cluster.local"
port: 9411
maxTagLength: 56 - name: "cloudtrace"
stackdriver:
maxTagLength: 256
{{< /text >}}

Istio のデフォルト設定には、すぐに使えるプロバイダーがいくつか含まれています：

| プロバイダー名 | 機能                                       |
| -------------- | ------------------------------------------ |
| `prometheus`   | メトリクス                                 |
| `stackdriver`  | メトリクス、分散トレーシング、アクセスログ |
| `envoy`        | アクセスログ                               |

また、[デフォルトプロバイダー](/zh/docs/reference/config/istio.mesh.v1alpha1/#MeshConfig-DefaultProviders)を設定しておくと、
`Telemetry` リソースでプロバイダーが指定されていない場合にこのデフォルトが使われます。

{{< tip >}}
[Sidecar](/zh/docs/reference/config/networking/sidecar/) 設定を使っている場合は、
プロバイダーのサービスを忘れずに追加してください。
{{< /tip >}}

{{< tip >}}
プロバイダーは `$(HOST_IP)` をサポートしません。コレクターをエージェントモードで動かす場合は、
[Service Internal Traffic Policy](https://kubernetes.io/zh-cn/docs/concepts/services-networking/service-traffic-policy/#using-service-internal-traffic-policy)
を使い、`InternalTrafficPolicy` を `Local` に設定するとパフォーマンスが向上します。
{{< /tip >}}

## 例 {#examples}

### メッシュ全体の動作を設定する {#configuring-mesh-wide-behavior}

Telemetry API リソースはメッシュのルート設定ネームスペース（通常は `istio-system`）から継承されます。
メッシュ全体の動作を設定するには、ルート設定ネームスペースに新しい（または既存の）`Telemetry` リソースを追加します。

以下は前節のプロバイダー設定を使ったサンプル設定です：

{{< text yaml >}}
apiVersion: telemetry.istio.io/v1
kind: Telemetry
metadata:
name: mesh-default
namespace: istio-system
spec:
tracing:

- providers: - name: localtrace
  customTags:
  foo:
  literal:
  value: bar
  randomSamplingPercentage: 100
  {{< /text >}}

この設定は `MeshConfig` 由来のデフォルトプロバイダーを上書きし、メッシュのデフォルトを `localtrace` プロバイダーにします。
また、メッシュ全体のサンプリング率を `100` に設定し、全ての span に `foo` という名前で `bar` という値のタグを追加します。

### ネームスペース単位のトレーシング動作を設定する {#configuring-namespace-scoped-tracing-behavior}

個別のネームスペースの動作をカスタマイズするには、ターゲットネームスペースに `Telemetry` リソースを追加します。
ネームスペースリソースで指定したフィールドは、設定階層から継承したフィールド設定を完全に上書きします。
例：

{{< text yaml >}}
apiVersion: telemetry.istio.io/v1
kind: Telemetry
metadata:
name: namespace-override
namespace: myapp
spec:
tracing:

- customTags:
  userId:
  header:
  name: userId
  defaultValue: unknown
  {{< /text >}}

先ほどのメッシュ全体のサンプル設定と組み合わせてデプロイすると、`myapp` ネームスペース内のトレーシング動作は、
span を `localtrace` プロバイダーに送信し、サンプリング率は `100%` ですが、
各 span には `userId` という名前でリクエストヘッダ `userId` の値をタグとして設定します。
重要なのは、`myapp` ネームスペースでは親設定の `foo: bar` タグは使われない点です。
カスタムタグの動作は `mesh-default.istio-system` リソースの設定を完全に上書きします。

{{< tip >}}
`Telemetry` リソースの全設定は、設定階層の親リソースの設定を完全に上書きします。これにはプロバイダー選択も含まれます。
{{< /tip >}}

### ワークロード単位の動作を設定する {#configuring-workload-specific-behavior}

個別のワークロードの動作をカスタマイズするには、ターゲットネームスペースに `Telemetry` リソースを追加し、`selector` を使います。
ワークロードリソースで指定したフィールドは、設定階層から継承したフィールド設定を完全に上書きします。

例：

{{< text yaml >}}
apiVersion: telemetry.istio.io/v1
kind: Telemetry
metadata:
name: workload-override
namespace: myapp
spec:
selector:
matchLabels:
service.istio.io/canonical-name: frontend
tracing:

- disableSpanReporting: true
  {{< /text >}}

この場合、`myapp` ネームスペースの `frontend` ワークロードではトレーシングが無効化されます。
Istio はトレーシングヘッダを転送し続けますが、span は設定されたトレーシングプロバイダーにレポートされません。

{{< tip >}}
ワークロードセレクタ付きの 2 つの `Telemetry` リソースが同じワークロードを選択するのは無効です。
この場合の動作は未定義です。
{{< /tip >}}
