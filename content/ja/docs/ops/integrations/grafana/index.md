---
title: Grafana
description: Grafana と統合して Istio ダッシュボードを構築する方法に関するドキュメント。
weight: 27
keywords: [integration, grafana]
owner: istio/wg-environments-maintainers
test: no
---

[Grafana](https://grafana.com/) はオープンソースの監視ソリューションで、Istio のダッシュボードを構成できます。Grafana を使って Istio やサービスメッシュ内のアプリケーションを監視できます。

## 設定 {#config}

独自のダッシュボードを作成することもできますが、Istio ではメッシュやコントロールプレーンの主要な指標を監視するための事前構成済みダッシュボードも提供しています。

- [メッシュダッシュボード](https://grafana.com/grafana/dashboards/7639) はメッシュ内のすべてのサービスの概要を表示します。
- [サービスダッシュボード](https://grafana.com/grafana/dashboards/7636) はサービスごとの詳細な指標を表示します。
- [ワークロードダッシュボード](https://grafana.com/grafana/dashboards/7630) はワークロードごとの詳細な指標を表示します。
- [パフォーマンスダッシュボード](https://grafana.com/grafana/dashboards/11829) はメッシュリソースの使用状況を監視します。
- [コントロールプレーンダッシュボード](https://grafana.com/grafana/dashboards/7645) はコントロールプレーンの健全性やパフォーマンス指標を監視します。
- [WASM 拡張ダッシュボード](https://grafana.com/grafana/dashboards/13277) は WebAssembly 拡張のランタイムやロード状況の概要を提供します。

これらのダッシュボードを Grafana で利用するには、いくつかの方法があります：

### 方法 1：クイックスタート {#option-1-quick-start}

Istio では、すべてのダッシュボードがバンドルされた基本的な Grafana インストール例を提供しています：

{{< text bash >}}
$ kubectl apply -f {{< github_file >}}/samples/addons/grafana.yaml
{{< /text >}}

`kubectl apply` で Grafana をクラスタにデプロイします。この方法はデモ用であり、パフォーマンスやセキュリティの調整はされていません。

### 方法 2：既存の Deployment へ grafana.com からインポート {#option-2-import-from-grafanacom-into-an-existing-deployment}

既存の Grafana インスタンスに Istio ダッシュボードを素早くインポートしたい場合は、[Grafana UI の **Import** ボタン](https://grafana.com/docs/grafana/latest/reference/export_import/#importing-a-dashboard)を使って上記ダッシュボードを追加できます。インポート時は Prometheus データソースを選択してください。

スクリプトで全ダッシュボードを一括インポートすることもできます。例：

{{< text bash >}}
$ # Grafana のアドレス
$ GRAFANA_HOST="http://localhost:3000"
$ # 認証が必要な場合のログイン情報
$ GRAFANA_CRED="USER:PASSWORD"
$ # 使用する Prometheus データソース名
$ GRAFANA_DATASOURCE="Prometheus"
$ # デプロイする Istio のバージョン
$ VERSION={{< istio_full_version >}}
$ # すべての Istio ダッシュボードをインポート
$ for DASHBOARD in 7639 11829 7636 7630 7645 13277; do
$ REVISION="$(curl -s https://grafana.com/api/dashboards/${DASHBOARD}/revisions -s | jq ".items[] | select(.description | contains(\"${VERSION}\")) | .revision" | tail -n 1)"
$ curl -s https://grafana.com/api/dashboards/${DASHBOARD}/revisions/${REVISION}/download > /tmp/dashboard.json
$ echo "Importing $(cat /tmp/dashboard.json | jq -r '.title') (revision ${REVISION}, id ${DASHBOARD})..."
$ curl -s -k -u "$GRAFANA_CRED" -XPOST \
$ -H "Accept: application/json" \
$ -H "Content-Type: application/json" \
$ -d "{\"dashboard\":$(cat /tmp/dashboard.json),\"overwrite\":true, \
$ \"inputs\":[{\"name\":\"DS_PROMETHEUS\",\"type\":\"datasource\", \
$ \"pluginId\":\"prometheus\",\"value\":\"$GRAFANA_DATASOURCE\"}]}" \
$ $GRAFANA_HOST/api/dashboards/import
$ echo -e "\nDone\n"
$ done
{{< /text >}}

### 方法 3：特定の実装方法 {#option-3-implementation-specific-methods}

Grafana の他のインストール・設定方法も利用できます。Istio ダッシュボードのインポート方法は、各種ドキュメントを参照してください：

- [Grafana のプロビジョニング](https://grafana.com/docs/grafana/latest/administration/provisioning/#dashboards)公式ドキュメント
- [ダッシュボードのインポート](https://github.com/helm/charts/tree/master/stable/grafana#import-dashboards) `stable/grafana` Helm Chart ドキュメント
