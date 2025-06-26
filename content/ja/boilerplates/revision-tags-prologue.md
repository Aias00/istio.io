---
---
リビジョン、ラベル、および名前空間間の結果のマッピングは次のとおりです：

{{< image width="90%"
link="/zh/docs/setup/upgrade/canary/revision-tags-after.svg"
caption="名前空間ラベルは変更されていないが、現在すべての名前空間が {{< istio_full_version_revision >}} を指しています"
>}}

`prod-stable` ラベル付き名前空間で注入ワークロードを再起動すると、これらのワークロードが `{{< istio_full_version_revision >}}` コントロールプレーンを使用するようになります。
名前空間の再ラベル付けは必要ありません。
