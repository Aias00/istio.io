---
title: Mixer アダプターモデル
description: Mixer のプラグインアーキテクチャの概要を説明します。
publishdate: 2017-11-03
subtitle: Istio とバックエンドインフラストラクチャの統合
attribution: Martin Taillefer
keywords: [adapters,mixer,policies,telemetry]
aliases:
    - /zh/blog/mixer-adapter-model.html
target_release: 0.2
---

Istio 0.2 では、新しい Mixer アダプターモデルが導入されました。このモデルは、バックエンドインフラストラクチャへの接続をより柔軟にします。この記事では、このモデルの仕組みを説明します。

## なぜアダプターモデルなのか？{#why-adapters}

バックエンドインフラストラクチャは、サービスの構築をサポートする機能を提供します。これらには、アクセス制御、テレメトリ、クォータ管理、課金システムなどがあります。従来のサービスは、これらのバックエンドシステムと直接統合され、バックエンドと密結合され、その中に個性化されたセマンティクスと操作が統合されています。

Mixer サービスは、Istio とオープンインフラストラクチャの間の抽象化レイヤーです。Istio コンポーネントとサービスメッシュで実行されるサービスは、バックエンドインターフェースに直接アクセスせずに、これらのバック

Mixer は、アプリケーション層とインフラストラクチャの間の隔離を提供するだけでなく、アプリケーションとバックエンドのポリシーを注入および制御するためのモデルを提供します。運用者は、どのデータをどのバックエンドに報告するか、どのバックエンドが認証を提供するかなどを制御できます。

各ベースサービスには、異なるインターフェースと操作モデルがあるため、Mixer はユーザーがコードでこれらの違いを解決する必要があります。これらのユーザーが独自にカプセル化したコードを [*アダプター*](https://github.com/istio/istio/wiki/Mixer-Compiled-In-Adapter-Dev-Guide) と呼びます。

アダプターは、Go パッケージの形式で Mixer バイナリに直接リンクされます。デフォルトのアダプターが特定の使用要件を満たさない場合、カスタムアダプターは非常に簡単に作成できます。

## 設計哲学{#philosophy}

Mixer は、属性とルーティングを処理する機械です。プロキシは、[属性](/zh/docs/reference/config/policy-and-telemetry/mixer-overview/#attributes)をプレチェックとテレメトリレポートの一部として送信し、それらを一連のアダプター呼び出しに変換します。運用者は、入力属性をアダプターにマッピングする方法を記述するための構成を提供します。

{{< image width="60%"
    link="/zh/blog/2017/adapter-model/machine.svg"
    caption="Attribute Machine"
    >}}

配置は複雑なタスクです。大多数のサービス中断は、設定エラーが原因であることが証明されています。この問題を解決するために、Mixer の設定モデルは、エラーを回避するために制限を設けています。例えば、設定で強い型を使用することで、コンテキスト環境で意味のある属性または属性式を使用することを確認します。

## Handlers: アダプターの設定{#handlers-configuring-adapters}

Mixer が使用する各アダプターは、実行するために構成が必要です。一般的に、アダプターはいくつかの情報が必要です。例えば、バックエンドの URL、証明書、キャッシュオプションなどです。各アダプターは、必要な構成データを定義するために [protobuf](https://developers.google.com/protocol-buffers/) メッセージを使用します。

[*handler*](/zh/docs/reference/config/policy-and-telemetry/mixer-overview/#handlers) を作成することで、アダプターの構成を提供できます。Handler は、アダプターを準備するための完全な構成です。同じアダプターに対して任意の数の Handler を作成できます。これにより、異なるシナリオで再利用できます。

## Templates: アダプターの入力構造{#templates-adapter-input-schema}

通常、メッシュサービスに入るリクエストでは、Mixer は 2 回呼び出されます。1 回はプレチェック、もう 1 回はテレメトリレポートです。各呼び出しで、Mixer は 1 つまたは複数のアダプターを呼び出します。異なるアダプターは、異なるデータを入力として必要とします。例えば、ログアダプターはログ入力を必要とし、メトリックアダプターはメトリックデータを必要とし、認証アダプターは証明書を必要とします。Mixer [*templates*](/zh/docs/reference/config/policy-and-telemetry/templates/) は、各リクエストでアダプターが消費するデータを記述するために使用されます。

各テンプレートは [protobuf](https://developers.google.com/protocol-buffers/) メッセージとして指定されます。テンプレートは、実行時に 1 つまたは複数のアダプターに渡されるデータのグループを記述します。アダプターは任意の数のテンプレートをサポートでき、開発者は特定のテンプレートをサポートするアダプターを設計できます。

[`metric`](/zh/docs/reference/config/policy-and-telemetry/templates/metric/) と [`logentry`](/zh/docs/reference/config/policy-and-telemetry/templates/logentry/) は、それぞれ負荷の単一指標と適切なバックエンドへの単一ログエントリを表す 2 つの最も重要なテンプレートです。

## Instances: 属性マッピング{#instances-attribute-mapping}

[*instances*](/zh/docs/reference/config/policy-and-telemetry/mixer-overview/#instances) を作成することで、特定のアダプターに送信されるデータを決定できます。インスタンスは、Mixer がプロキシからの属性をさまざまなデータに分割し、それらを異なるアダプターに配布する方法を決定します。

インスタンスを作成する通常は、[attribute expressions](/zh/docs/reference/config/policy-and-telemetry/expression-language/) を使用する必要があります。これらの式の機能は、属性と定数を使用して結果データを生成し、instance フィールドに値を割り当てることです。

各インスタンスフィールド、各属性、各式には [type](https://github.com/istio/api/blob/{{< source_branch_name >}}/policy/v1beta1/value_type.proto) があり、互換性のあるデータ型のみを割り当てることができます。例えば、整数式を文字列型に割り当てることはできません。強い型付け設計の目的は、設定エラーが原因でサービス中断が発生するリスクを低減することです。

## Rules: アダプターへのデータ配信{#rules-delivering-data-to-adapters}

最後の問題は、Mixer にどのインスタンスをどのハンドラーにいつ送信するかを指示することです。これは [*rules*](/zh/docs/reference/config/policy-and-telemetry/mixer-overview/#rules) を作成することで実現できます。各ルールは、特定のハンドラーとそのハンドラーに送信されるインスタンスを指定します。Mixer が呼び出しを処理するとき、指定されたハンドラーを呼び出し、特定のインスタンスのセットを提供します。

Rule にはマッチングアサーションが含まれており、これは属性式であり、ブール値を返します。属性式アサーションが成功した Rule のみが有効になります。そうでない場合、ルールは無効になり、その中の Handler も呼び出されません。

## 今後の課題{#future}

アダプターの使用と開発を改善および向上させるために努力しています。例えば、テンプレートの使用をより便利にするために多くの新機能を計画しています。また、式言語も発展し、成熟しています。

長期的には、アダプターを Mixer バイナリに直接接続する方法を見つけることを目指しています。これにより、デプロイと開発がより簡単になります。

## まとめ{#conclusion}

新しい Mixer アダプターモデルの設計は、オープンインフラストラクチャをサポートするための柔軟なフレームワークを提供することを目的としています。

Handler は各アダプターの構成データを提供し、Template は実行時に異なるアダプターが必要とするデータ型を決定し、Instance は運用者がこれらのデータを準備し、Rule はこれらのデータを 1 つまたは複数の Handler に送信して処理します。

詳細については、[ここ](/zh/docs/reference/config/policy-and-telemetry/mixer-overview/) を参照してください。テンプレート、ハンドラー、およびルールについての詳細は、[ここ](/zh/docs/reference/config/policy-and-telemetry/) を参照してください。また、[ここ]({{<github_tree>}}/samples/bookinfo) で対応するサンプルを確認できます。
