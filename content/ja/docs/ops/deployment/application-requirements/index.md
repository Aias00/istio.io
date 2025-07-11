---
title: アプリケーション要件
description: Istio 対応クラスタにデプロイするアプリケーションの要件。
weight: 40
keywords:
  - kubernetes
  - sidecar
  - sidecar-injection
  - deployment-models
  - pods
  - setup
aliases:
  - /zh/docs/setup/kubernetes/spec-requirements/
  - /zh/docs/setup/kubernetes/prepare/spec-requirements/
  - /zh/docs/setup/kubernetes/prepare/requirements/
  - /zh/docs/setup/kubernetes/additional-setup/requirements/
  - /zh/docs/setup/additional-setup/requirements
  - /zh/docs/ops/setup/required-pod-capabilities
  - /zh/help/ops/setup/required-pod-capabilities
  - /zh/docs/ops/prep/requirements
  - /zh/docs/ops/deployment/requirements
owner: istio/wg-environments-maintainers
test: n/a
---

Istio はアプリケーションコード自体にほとんど影響を与えず、多くの機能を提供します。
多くの Kubernetes アプリケーションは、Istio 有効クラスタに修正なしでデプロイできます。
ただし、Istio Sidecar モデルの影響には注意が必要です。
この記事では、これらのアプリケーションに関する注意点と、Istio 有効化のための要件を説明します。

## Pod 要件 {#pod-requirements}

Istio サービスメッシュの一部として、Kubernetes クラスタ内の Pod および Service は以下の要件を満たす必要があります：

- **アプリケーション UID**：Pod で UID `1337` のユーザーでアプリケーションを実行しないでください。`1337` は Sidecar プロキシ用に予約されています。

- **`NET_ADMIN` および `NET_RAW` 権限**：クラスタで[Pod セキュリティポリシー](https://kubernetes.io/ja/docs/concepts/policy/pod-security-policy/)が[強制されている](https://kubernetes.io/ja/docs/concepts/policy/pod-security-policy/#enabling-pod-security-policies)場合、Pod に `NET_ADMIN` および `NET_RAW` 権限を付与する必要があります。[Istio CNI プラグイン](/ja/docs/setup/additional-setup/cni/)を使用している場合は不要です。

  Pod に `NET_ADMIN` および `NET_RAW` 権限があるか確認するには、Pod の[サービスアカウント](https://kubernetes.io/ja/docs/tasks/configure-pod-container/configure-service-account/)がこれらの権限を持つ Pod セキュリティポリシーを持っているか確認します。Pod デプロイでサービスアカウントを指定していない場合、Pod はネームスペースのデフォルトサービスアカウントで実行されます。

  サービスアカウントの権限を一覧表示するには、以下のコマンドで `<your namespace>` と `<your service account>` を置き換えてください。

  {{< text bash >}}
  $ for psp in $(kubectl get psp -o jsonpath="{range .items[*]}{@.metadata.name}{'\n'}{end}"); do if [ $(kubectl auth can-i use psp/$psp --as=system:serviceaccount:<your namespace>:<your service account>) = yes ]; then kubectl get psp/$psp --no-headers -o=custom-columns=NAME:.metadata.name,CAPS:.spec.allowedCapabilities; fi; done
  {{< /text >}}

  例：`default` ネームスペースの `default` サービスアカウントを確認するには：

  {{< text bash >}}
  $ for psp in $(kubectl get psp -o jsonpath="{range .items[*]}{@.metadata.name}{'\n'}{end}"); do if [ $(kubectl auth can-i use psp/$psp --as=system:serviceaccount:default:default) = yes ]; then kubectl get psp/$psp --no-headers -o=custom-columns=NAME:.metadata.name,CAPS:.spec.allowedCapabilities; fi; done
  {{< /text >}}

  サービスアカウントの許可ポリシーの機能リストに `NET_ADMIN`、`NET_RAW`、または `*` があれば、Pod は Istio Init コンテナを実行できます。なければ[権限を付与](https://kubernetes.io/ja/docs/concepts/security/pod-security-policy)してください。

- **Pod ラベル**：Pod にはアプリケーション識別子やバージョンを明示するラベルを付与することを推奨します。
  これらのラベルは、Istio が収集するメトリクスやテレメトリーにコンテキスト情報を追加します。
  各値は複数のラベルから優先順位順に取得されます：

  - アプリ名：`service.istio.io/canonical-name`、`app.kubernetes.io/name`、`app`
  - アプリバージョン：`service.istio.io/canonical-revision`、`app.kubernetes.io/version`、`version`

- **名前付き Service ポート**：Service ポートに名前を付けてプロトコルを明示指定できます。
  詳細は[プロトコル選択](/ja/docs/ops/configuration/traffic-management/protocol-selection/)を参照してください。
  1 つの Pod が複数の [Kubernetes Service](https://kubernetes.io/ja/docs/concepts/services-networking/service/) に属する場合、異なるプロトコル（例：HTTP と TCP）で同じポート番号を使うことはできません。

## Istio が使用するポート {#ports-used-by-Istio}

Istio Sidecar プロキシ（Envoy）は以下のポートとプロトコルを使用します。

{{< warning >}}
Sidecar とのポート競合を避けるため、アプリケーションは Envoy が使用するポートを利用しないでください。
{{< /warning >}}

| ポート | プロトコル | 説明                                                                  | Pod 内部のみ |
| ------ | ---------- | --------------------------------------------------------------------- | ------------ |
| 15000  | TCP        | Envoy 管理ポート（コマンド/診断）                                     | はい         |
| 15001  | TCP        | Envoy アウトバウンド                                                  | いいえ       |
| 15002  | TCP        | ヘルスチェックリスナーポート                                          | はい         |
| 15004  | HTTP       | デバッグポート                                                        | はい         |
| 15006  | TCP        | Envoy インバウンド                                                    | いいえ       |
| 15008  | H2         | HBONE mTLS トンネルポート                                             | いいえ       |
| 15020  | HTTP       | Istio プロキシ、Envoy、アプリケーションからの Prometheus テレメトリー | いいえ       |
| 15021  | HTTP       | ヘルスチェック                                                        | いいえ       |
| 15053  | DNS        | DNS ポート（キャプチャ有効時）                                        | はい         |
| 15090  | HTTP       | Envoy Prometheus テレメトリー                                         | いいえ       |

Istio コントロールプレーン（istiod）は以下のポートとプロトコルを使用します。

| ポート | プロトコル | 説明                                                               | ローカルホストのみ |
| ------ | ---------- | ------------------------------------------------------------------ | ------------------ |
| 443    | HTTPS      | Webhook サービス用ポート                                           | いいえ             |
| 8080   | HTTP       | デバッグインターフェース（非推奨、コンテナポートのみ）             | いいえ             |
| 15010  | GRPC       | XDS および CA サービス（プレーンテキスト、安全なネットワークのみ） | いいえ             |
| 15012  | GRPC       | XDS および CA サービス（TLS/mTLS、本番推奨）                       | いいえ             |
| 15014  | HTTP       | コントロールプレーン監視                                           | いいえ             |
| 15017  | HTTPS      | Webhook コンテナポート（443 から転送）                             | いいえ             |

## サーバーファーストプロトコル {#server-first-protocols}

一部のプロトコルは「サーバーファースト」プロトコルで、サーバーが最初のバイトを送信します。これは、
[`PERMISSIVE`](/ja/docs/reference/config/security/peer_authentication/#PeerAuthentication-MutualTLS-Mode)
mTLS や[自動プロトコル選択](/ja/docs/ops/configuration/traffic-management/protocol-selection/#automatic-protocol-selection)に影響します。

これらの機能はどちらも、接続の最初のバイトを検査してプロトコルを判定しますが、サーバーファーストプロトコルとは互換性がありません。

このような場合は、
[明示的なプロトコル選択](/ja/docs/ops/configuration/traffic-management/protocol-selection/#explicit-protocol-selection)の手順に従い、アプリケーションのプロトコルを `TCP` として宣言してください。

以下のポートはサーバーファーストプロトコルであることが多く、自動的に `TCP` とみなされます：

| プロトコル | ポート |
| ---------- | ------ |
| SMTP       | 25     |
| DNS        | 53     |
| MySQL      | 3306   |
| MongoDB    | 27017  |

TLS 通信はサーバーファーストではないため、TLS で暗号化されたサーバーファーストトラフィックは自動プロトコル検出と併用できます。
TLS スニッフィング対象のトラフィックがすべて暗号化されていることを確認してください：

1. サーバーの `mTLS` モードを `STRICT` に設定します。これによりすべてのリクエストで TLS 暗号化が強制されます。
1. サーバーの `mTLS` モードを `DISABLE` に設定します。これにより TLS スニッフィングが無効化され、サーバーファーストプロトコルが許可されます。
1. すべてのクライアントが `TLS` トラフィックを送信するよう設定します（通常は
   [`DestinationRule`](/ja/docs/reference/config/networking/destination-rule/#ClientTLSSettings)
   または自動 mTLS を利用）。
1. アプリケーションを直接 TLS トラフィック送信に設定します。

## アウトバウンドトラフィック {#outbound-traffic}

Istio のトラフィックルーティング機能をサポートするため、Pod から出るトラフィックは Sidecar 未導入時と異なる場合があります。

HTTP ベースのトラフィックは `Host` ヘッダーでルーティングされます。宛先 IP と `Host`
ヘッダーが一致しない場合、予期しない動作になることがあります。たとえば、`curl 1.2.3.4 -H "Host: httpbin.default"`
は `1.2.3.4` ではなく `httpbin` サービスにルーティングされます。

HTTP 以外のトラフィック（HTTPS など）では、Istio は `Host` ヘッダーにアクセスできないため、サービス IP アドレスでルーティングします。

つまり、Pod を直接呼び出す（例：`curl <POD_IP>`）場合、Service にはマッチしません。
この場合、トラフィックは[パススルー](/ja/docs/tasks/traffic-management/egress/egress-control/#envoy-passthrough-to-external-services)されますが、mTLS 暗号化やトラフィックルーティング、テレメトリーなど Istio の機能は利用できません。

詳細は[トラフィックルーティング](/ja/docs/ops/configuration/traffic-management/traffic-routing)ページを参照してください。
