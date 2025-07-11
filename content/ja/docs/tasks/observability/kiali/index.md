---
title: メッシュの可視化
description: このタスクでは、Istio メッシュ内のサービスを可視化する方法を紹介します。
weight: 49
keywords: [telemetry, visualization]
aliases:
  - /zh/docs/tasks/telemetry/kiali/
owner: istio/wg-policies-and-telemetry-maintainers
test: no
---

このタスクでは、Istio メッシュのさまざまな側面を可視化する方法を紹介します。

このタスクの一環として [Kiali](https://www.kiali.io) アドオンをインストールし、
Web ベースのグラフィカルユーザーインターフェースを使ってメッシュや Istio 構成オブジェクトのサービスグラフを確認します。

{{< idea >}}
このタスクは Kiali のすべての機能を網羅しているわけではありません。サポートされている全機能については [Kiali 公式サイト](http://kiali.io/docs/features/) をご覧ください。
{{< /idea >}}

このタスクでは常に [Bookinfo](/zh/docs/examples/bookinfo/) サンプルアプリケーションを例として使用します。
Bookinfo アプリケーションは `bookinfo` 名前空間にインストールされているものとします。

## 始める前に {#before-you-begin}

[Kiali インストール](/zh/docs/ops/integrations/kiali/#installation)ドキュメントに従い、Kiali をクラスタにデプロイしてください。

## サービスグラフの生成 {#generating-a-graph}

1. クラスタ内でサービスが稼働していることを確認するには、次のコマンドを実行します：

   {{< text bash >}}
   $ kubectl -n istio-system get svc kiali
   {{< /text >}}

1. Bookinfo の URL を特定するには、
   [Bookinfo ingress `GATEWAY_URL`](/zh/docs/examples/bookinfo/#determine-the-ingress-IP-and-port) の手順に従ってください。

1. メッシュにトラフィックを送るには、次のいずれかの方法を利用できます：

   - ブラウザで `http://$GATEWAY_URL/productpage` にアクセス

   - 以下のコマンドを複数回実行：

     {{< text bash >}}
     $ curl http://$GATEWAY_URL/productpage
     {{< /text >}}

   - システムに `watch` コマンドがインストールされている場合、次のコマンドでリクエストを連続送信：

     {{< text bash >}}
     $ watch -n 1 curl -o /dev/null -s -w %{http_code} $GATEWAY_URL/productpage
     {{< /text >}}

1. Kubernetes 環境で Kiali UI を開くには、次のコマンドを実行します：

   {{< text bash >}}
   $ istioctl dashboard kiali
   {{< /text >}}

1. ログイン直後の **Overview** ページでメッシュの概要を確認します。
   **Overview** ページにはメッシュ内でサービスを持つすべての名前空間が表示されます。以下はその例です：

   {{< image width="75%"
        link="./kiali-overview.png"
        caption="Overview の例"
        >}}

1. 名前空間のグラフを表示するには、Bookinfo 名前空間カードの `Graph` メニュー項目を選択します。
   ケバブメニューはカード右上の 3 点アイコンです。
   クリックすると利用可能なメニュー項目が表示されます。以下のようになります：

   {{< image width="75%"
        link="./kiali-graph.png"
        caption="Graph の例"
        >}}

1. このグラフは、一定期間にサービスメッシュを流れるトラフィックを表します。グラフは Istio テレメトリを使って生成されます。

1. 指標サマリーを確認するには、グラフ内の任意のノードまたはエッジを選択し、右側の summary details パネルでその指標の詳細を表示します。

1. **Graph Type** ドロップダウンから異なるグラフタイプを選択してサービスメッシュを可視化できます。
   選択可能なグラフタイプは **App**、**Versioned App**、**Workload**、**Service** です。

   - **App** グラフタイプは、アプリケーションのすべてのバージョンを 1 つのノードに集約します。
     以下は 3 つのバージョンを持つ **reviews** ノードの例です。

     {{< image width="75%"
           link="./kiali-app.png"
           caption="アプリケーショングラフの例"
           >}}

   - **Versioned App** グラフタイプは、各アプリケーションバージョンのノードを表示しますが、同じアプリケーションのすべてのバージョンをまとめて表示します。
     以下は 3 つのバージョンを持つ **reviews** グループボックスの例です。

     {{< image width="75%"
           link="./kiali-versionedapp.png"
           caption="バージョン付きアプリケーショングラフの例"
           >}}

   - **Workload** グラフタイプは、サービスメッシュ内の各ワークロードのノードを表示します。
     このグラフタイプは `app` や `version` ラベルがなくても利用できます。
     コンポーネントにこれらのラベルを付与しない場合はこのタイプを使います。

     {{< image width="70%"
           link="./kiali-workload.png"
           caption="ワークロードグラフの例"
           >}}

   - **Service** グラフタイプは、メッシュ内のサービス間の高レベルなトラフィックを表示します。

     {{< image width="70%"
     link="./kiali-service-graph.png"
     caption="サービスグラフの例"

     > }}

## Istio 設定の確認 {#examining-Istio-configuration}

1. Istio 設定の詳細を確認するには、左側メニューの **Applications**、**Workloads**、**Services** をクリックします。
   以下は Bookinfo アプリケーション情報の例です：

   {{< image width="80%"
       link="./kiali-services.png"
       caption="詳細の例"
       >}}

## トラフィックシフト {#traffic-shifting}

Kiali のトラフィックシフトウィザードを使うと、リクエストの特定の割合を 2 つ以上のワークロードにルーティングできます。

1. `bookinfo` グラフの **Versioned app graph** を表示します。

   - **Traffic Distribution Edge Label** の **Display** オプションを有効にして、各ワークロードへのトラフィック割合を確認します。

   - **Show Service Nodes** の **Display** オプションを有効にして、グラフにサービスノードを表示します。

   {{< image width="80%"
       link="./kiali-wiz0-graph-options.png"
       caption="Bookinfo グラフオプション"
       >}}

1. `ratings` サービス（三角形ノード）をクリックして、`bookinfo` グラフ内の `ratings` サービスに注目します。
   `ratings` サービスのトラフィックが `ratings-v1` と `ratings-v2` に均等（50%ずつ）に分配されていることを確認します。

   {{< image width="80%"
       link="./kiali-wiz1-graph-ratings-percent.png"
       caption="トラフィック割合表示グラフ"
       >}}

1. サイドパネルの **ratings** リンクをクリックして `ratings` サービスの詳細ビューに移動します。
   または `ratings` サービスノードを右クリックし、コンテキストメニューから `Details` を選択しても移動できます。

1. **Action** ドロップダウンから **Traffic Shifting** を選択してトラフィックシフトウィザードを開きます。

   {{< image width="80%"
       link="./kiali-wiz2-ratings-service-action-menu.png"
       caption="サービスのアクションメニュー"
       >}}

1. スライダーを動かして各サービスへのトラフィック割合を指定します。
   `ratings-v1` を 10%、`ratings-v2` を 90% に設定します。

   {{< image width="80%"
       link="./kiali-wiz3-traffic-shifting-wizard.png"
       caption="重み付きルーティングウィザード"
       >}}

1. **Preview** ボタンをクリックしてウィザードが生成する YAML を確認します。

   {{< image width="80%"
       link="./kiali-wiz3b-traffic-shifting-wizard-preview.png"
       caption="ルーティングウィザードプレビュー"
       >}}

1. **Create** ボタンをクリックして新しいトラフィック設定を適用します。

1. 左側ナビゲーションの **Graph** をクリックして `bookinfo` グラフに戻ります。`ratings` サービスノードに `virtual service` アイコンが付いていることを確認します。

1. `bookinfo` アプリケーションにリクエストを送信します。例えば、1 秒ごとにリクエストを送るには、システムに `watch` があれば次のコマンドを実行します：

   {{< text bash >}}
   $ watch -n 1 curl -o /dev/null -s -w %{http_code} $GATEWAY_URL/productpage
   {{< /text >}}

1. 数分後、トラフィック割合が新しいルーティングを反映し、全リクエストの 90%が `ratings-v2` にルーティングされていることを確認できます。

   {{< image width="80%"
       link="./kiali-wiz4-traffic-shifting-90-10.png"
       caption="90% Ratings トラフィックが ratings-v2 へ"
       >}}

## Istio 設定のバリデーション {#validating-Istio-configuration}

Kiali は Istio リソースを検証し、正しい規約やセマンティクスに従っているかをチェックできます。
設定の重大度に応じて、検出された問題はエラーまたは警告としてマークされます。
Kiali が実行するすべてのバリデーションチェックの一覧は [Kiali Validation ページ](https://kiali.io/docs/features/validations/) を参照してください。

{{< idea >}}
Istio には `istioctl analyze` もあり、CI パイプラインなどで同様の分析が可能です。両者は補完的に利用できます。
{{< /idea >}}

サービスのポート名を無効な値に変更し、Kiali がどのようにバリデーションエラーを報告するかを確認します。

1. `details` サービスのポート名を `http` から `foo` に変更します：

   {{< text bash >}}
   $ kubectl patch service details -n bookinfo --type json -p '[{"op":"replace","path":"/spec/ports/0/name", "value":"foo"}]'
   {{< /text >}}

1. 左側ナビゲーションの **Services** をクリックして **Services** 一覧に移動します。

1. **Namespace** ドロップダウンから `bookinfo` を選択していない場合は選択します。

1. `details` 行の **Configuration** 列にエラーアイコンが表示されていることを確認します。

   {{< image width="80%"
       link="./kiali-validate1-list.png"
       caption="無効な設定を示すサービス一覧"
       >}}

1. **Name** 列の **details** リンクをクリックしてサービス詳細ビューに移動します。

1. エラーアイコンにマウスを重ねると、エラー内容のツールチップが表示されます。

   {{< image width="80%"
       link="./kiali-validate2-errormsg.png"
       caption="無効な設定を示すサービス詳細"
       >}}

1. ポート名を `http` に戻して設定を修正し、`bookinfo` を正常な状態に戻します。

   {{< text bash >}}
   $ kubectl patch service details -n bookinfo --type json -p '[{"op":"replace","path":"/spec/ports/0/name", "value":"http"}]'
   {{< /text >}}

   {{< image width="80%"
       link="./kiali-validate3-ok.png"
       caption="無効な設定を示すサービス詳細"
       >}}

## Istio YAML 設定ファイルの閲覧と編集 {#viewing-and-editing-Istio-configuration-YAML}

Kiali には Istio 設定リソースの閲覧・編集用 YAML エディタがあり、エラー検出時にはバリデーションメッセージも表示されます。

1. `bookinfo` VirtualService にエラーを導入します。

   {{< text bash >}}
   $ kubectl patch vs bookinfo -n bookinfo --type json -p '[{"op":"replace","path":"/spec/gateways/0", "value":"bookinfo-gateway-invalid"}]'
   {{< /text >}}

1. 左側ナビゲーションの `Istio Config` をクリックして Istio 設定一覧に移動します。

1. **Namespace** ドロップダウンから `bookinfo` を選択していない場合は選択します。

1. エラーメッセージや警告アイコンが表示され、設定に問題があることが分かります。

   {{< image width="80%"
       link="./kiali-istioconfig0-errormsgs.png"
       caption="Istio Config のエラー一覧"
       >}}

1. `bookinfo` 行の **Configuration** 列のエラーアイコンをクリックして `bookinfo` VirtualService 詳細ビューに移動します。

1. **YAML** タブが選択されていることを確認します。バリデーション通知が関連行の色やアイコンで強調表示されます。

   {{< image width="80%"
       link="./kiali-istioconfig3-details-yaml1.png"
       caption="YAML エディタのバリデーション通知"
       >}}

1. 赤いアイコンにマウスを重ねると、バリデーションエラーのツールチップが表示されます。
   エラーの原因や解決方法の詳細は [Kiali Validation ページ](https://kiali.io/docs/features/validations/) を参照してください。

   {{< image width="80%"
       link="./kiali-istioconfig3-details-yaml3.png"
       caption="YAML エディタのエラーツールチップ"
       >}}

1. VirtualService `bookinfo` を元の状態にリセットします。

   {{< text bash >}}
   $ kubectl patch vs bookinfo -n bookinfo --type json -p '[{"op":"replace","path":"/spec/gateways/0", "value":"bookinfo-gateway"}]'
   {{< /text >}}

## その他の機能 {#additional-features}

本記事で紹介した閲覧機能以外にも、Kiali には[Jaeger トレース統合](https://kiali.io/docs/features/tracing/)など多くの機能があります。

詳細は [Kiali ドキュメント](https://kiali.io/docs/features/) をご覧ください。

Kiali を深く学びたい場合は [Kiali チュートリアル](https://kiali.io/docs/tutorials/) の実践をおすすめします。

## クリーンアップ {#cleanup}

今後のタスクを試す予定がなければ、Bookinfo サンプルアプリケーションと Kiali をクラスタから削除してください。

1. Bookinfo アプリケーションを削除するには、[Bookinfo のクリーンアップ](/zh/docs/examples/bookinfo/#cleanup)の手順に従ってください。

1. Kubernetes 環境から Kiali を削除するには：

   {{< text bash >}}
   $ kubectl delete -f {{< github_file >}}/samples/addons/kiali.yaml
   {{< /text >}}
