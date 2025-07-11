---
title: オブザーバビリティの問題
description: テレメトリ収集の問題への対応。
force_inline_toc: true
weight: 30
aliases:
  - /zh/docs/ops/troubleshooting/grafana
  - /zh/docs/ops/troubleshooting/missing-traces
owner: istio/wg-policies-and-telemetry-maintainers
test: n/a
---

## Mac で Istio を実行しているときに Zipkin にトレースが表示されない {#no-traces-appearing-in-Zipkin-when-running-Istio-locally-on-mac}

Istio のインストールは完了し正常に動作しているが、Zipkin にトレース情報が表示されない場合があります。

これは既知の [Docker の問題](https://github.com/docker/for-mac/issues/1260)が原因で、
コンテナ内の時刻がホストの時刻と大きくずれている可能性があります。この場合、Zipkin で長い時間範囲を選択すると、
データが予想よりも早い時刻に記録されていることが分かるかもしれません。

また、Docker コンテナ内と外で日付を比較することでこの問題を確認できます：

{{< text bash >}}
$ docker run --entrypoint date gcr.io/istio-testing/ubuntu-16-04-slave:latest
Sun Jun 11 11:44:18 UTC 2017
{{< /text >}}

{{< text bash >}}
$ date -u
Thu Jun 15 02:25:42 UTC 2017
{{< /text >}}

この問題を解決するには、Docker を再起動し、その後 Istio を再インストールしてください。

## Grafana の出力が表示されない {#missing-Grafana-output}

ローカルの Web クライアントから Istio に接続した際に Grafana のデータが取得できない場合、
クライアントとサーバーの日時が一致しているか確認してください。

Web クライアント（例：Chrome）の時刻は Grafana の出力に影響します。簡単な解決策は、
Kubernetes クラスタ内の時刻同期サービスが正しく動作していること、
Web クライアントのコンピュータも対象サーバーと同じ時刻になっていることを確認することです。
よく使われる時刻同期システムには NTP や Chrony があります。ファイアウォールのあるラボ環境では、
NTP 設定が誤っていてラボ内の NTP サーバーを正しく参照できていない場合にこの問題が発生しやすいです。

## Istio CNI Pod が稼働しているか確認する（利用している場合） {#verify-Istio-CNI-pods-are-running}

Istio CNI プラグインは Kubernetes Pod ライフサイクルのネットワーク設定段階で Istio メッシュ Pod のトラフィックリダイレクトを実行し、
ユーザーが Istio メッシュに Pod をデプロイする際の [`NET_ADMIN` および `NET_RAW` 権限の依存](/zh/docs/ops/deployment/application-requirements/)を排除します。
Istio CNI プラグインは `istio-init` コンテナの機能を置き換えます。

1. `istio-cni-node` Pod が稼働しているか確認します：

   {{< text bash >}}
   $ kubectl -n kube-system get pod -l k8s-app=istio-cni-node
   {{< /text >}}

1. クラスタで `PodSecurityPolicy` が有効な場合、`istio-cni` Service Account が `PodSecurityPolicy` の
   [`NET_ADMIN` および `NET_RAW` 機能](/zh/docs/ops/deployment/application-requirements/)を利用できることを確認してください。
