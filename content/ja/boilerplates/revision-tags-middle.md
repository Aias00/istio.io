---
---
リビジョン、ラベル、および名前空間間の結果のマッピングは次のとおりです：

{{< image width="90%"
link="/zh/docs/setup/upgrade/canary/revision-tags-before.svg"
caption="2 つの名前空間が prod-stable を指し、1 つが prod-canary を指しています"
>}}

ラベル付き名前空間以外のクラスター管理者は、以下の `istioctl tag list` コマンドを使用してこのマッピングを確認できます：

{{< text bash >}}
$ istioctl tag list
TAG         REVISION NAMESPACES
default     {{< istio_previous_version_revision >}}-1   ...
prod-canary {{< istio_full_version_revision >}}   ...
prod-stable {{< istio_previous_version_revision >}}-1   ...
{{< /text >}}

クラスター管理者が `prod-canary` とラベル付けされたコントロールプレーンの名前空間の安定性に満足したら、
`istio.io/rev=prod-stable` を更新して、新しい `{{< istio_full_version_revision >}}` リビジョンを指すようにすることができます。
