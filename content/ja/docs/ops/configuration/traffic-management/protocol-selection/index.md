---
title: プロトコル選択
description: プロトコルの宣言方法について。
weight: 10
keywords: [protocol, protocol sniffing, protocol selection, protocol detection]
aliases:
  - /zh/help/ops/traffic-management/protocol-selection
  - /zh/help/ops/protocol-selection
  - /zh/help/tasks/traffic-management/protocol-selection
  - /zh/docs/ops/traffic-management/protocol-selection
  - /zh/docs/ops/deployment/requirements
owner: istio/wg-networking-maintainers
test: no
---

Istio はデフォルトで、すべての TCP トラフィック（HTTP、HTTPS、gRPC、純粋な TCP プロトコルを含む）をプロキシできます。しかし、追加の機能（ルーティングや豊富なメトリクスなど）を提供するには、プロトコルを特定する必要があります。プロトコルは自動検出または手動で宣言できます。

TCP ベースでないプロトコル（例：UDP）は Istio プロキシでインターセプトされず、通常通り動作します。
ただし、Ingress や Egress Gateway などプロキシのみのコンポーネントでは使用できません。

## 自動プロトコル選択 {#automatic-protocol-selection}

Istio は HTTP および HTTP/2 トラフィックを自動的に検出できます。プロトコルが自動検出されない場合、トラフィックは通常の TCP トラフィックとして扱われます。

{{< tip >}}
Server First プロトコル（MySQL など）は自動プロトコル選択と互換性がありません。
詳細は [Server First プロトコル](/ja/docs/ops/deployment/application-requirements#server-first-protocols) を参照してください。
{{< /tip >}}

## 明示的なプロトコル選択 {#explicit-protocol-selection}

プロトコルは Service 定義で手動指定できます。

設定方法は 2 通りあります：

- ポート名で指定：`name: <protocol>[-<suffix>]`
- Kubernetes 1.18+ では `appProtocol` フィールドで指定：`appProtocol: <protocol>`

両方が定義されている場合、`appProtocol` が優先されます。

なお、Gateway では TLS 終端やプロトコルネゴシエーションが発生するため、挙動が異なる場合があります。

サポートされるプロトコル：

| プロトコル                | Sidecar 用途                                                                                                                                            | Gateway 用途                                                                                                                                            |
| ------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `http`                    | HTTP/1.1 プレーンテキストトラフィック                                                                                                                   | HTTP（1.1 または 2）プレーンテキストトラフィック                                                                                                        |
| `http2`                   | HTTP/2 プレーンテキストトラフィック                                                                                                                     | HTTP（1.1 または 2）プレーンテキストトラフィック                                                                                                        |
| `https`                   | TLS で暗号化されたデータ。Sidecar では TLS を復号しないため `tls` と同じ扱い。                                                                          | TLS で暗号化された HTTP（1.1 または 2）トラフィック                                                                                                     |
| `tcp`                     | 透過的な TCP データストリーム                                                                                                                           | 透過的な TCP データストリーム                                                                                                                           |
| `tls`                     | TLS で暗号化されたデータ                                                                                                                                | TLS で暗号化されたデータ                                                                                                                                |
| `grpc`、`grpc-web`        | `http2` と同じ                                                                                                                                          | `http2` と同じ                                                                                                                                          |
| `mongo`、`mysql`、`redis` | 実験的なアプリケーションプロトコルサポート。利用には[環境変数](/ja/docs/reference/commands/pilot-discovery/#envvars)の設定が必要。未設定時は TCP 扱い。 | 実験的なアプリケーションプロトコルサポート。利用には[環境変数](/ja/docs/reference/commands/pilot-discovery/#envvars)の設定が必要。未設定時は TCP 扱い。 |

以下の例は、`appProtocol` で `https` ポート、`name` で `http` ポートを定義した Service です：

{{< text yaml >}}
kind: Service
metadata:
name: myservice
spec:
ports:

- port: 3306
  name: database
  appProtocol: mysql
- port: 80
  name: http-web
  {{< /text >}}

## HTTP Gateway プロトコル選択 {#http-gateway-protocol-selection}

Sidecar と異なり、Gateway はデフォルトでバックエンドサービスへの転送時に使用する HTTP プロトコルを自動検出できません。
そのため、明示的なプロトコル選択で HTTP/1.1（`http`）や HTTP/2（`http2` または `grpc`）を指定しない限り、
Gateway はすべての受信 HTTP リクエストを HTTP/1.1 で転送します。

明示的なプロトコル選択のほか、サービスに [`useClientProtocol`](/ja/docs/reference/config/networking/destination-rule/#ConnectionPoolSettings-HTTPSettings) オプションを設定することで、Gateway に受信リクエストと同じプロトコルで転送させることもできます。
ただし、HTTP/2 非対応サービスでこのオプションを使うと問題が発生する場合があります。
HTTPS Gateway は常に[ALPN](https://en.wikipedia.org/wiki/Application-Layer_Protocol_Negotiation) で HTTP/1.1 と HTTP/2 の両方をアドバタイズするため、
最新のクライアントは後者を選択しがちですが、バックエンドが HTTP/2 非対応だと通信できなくなります。
