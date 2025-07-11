---
title: DNS を理解する
linktitle: DNS
description: Istio が DNS とどのように連携するかを理解する。
weight: 31
keywords: [traffic-management, proxy]
owner: istio/wg-networking-maintainers
test: n/a
---

Istio は DNS とさまざまな方法で連携しますが、これは混乱を招くことがあります。本ドキュメントでは、
Istio と DNS の連携方法について詳しく説明します。

{{< warning >}}

このドキュメントは低レベルの実装詳細を説明しています。より高レベルの概要については、
トラフィック管理の[コンセプト](/ja/docs/concepts/traffic-management/)や
[タスク](/ja/docs/tasks/traffic-management/)ページをご覧ください。

{{< /warning >}}

## リクエストの流れ {#life-of-a-request}

ここでは、アプリケーションが `curl example.com` を実行する例を使って、リクエストの全体的な流れを見ていきます。
この `curl` のリクエストフローは、ほとんどすべてのクライアントに当てはまります。

ドメイン名にリクエストを送信すると、クライアントは DNS 解決を行い、IP アドレスに変換します。
Istio の設定に関係なく、これは必ず発生します。Istio はネットワークトラフィックをインターセプトするだけで、
アプリケーションの動作や DNS リクエストの有無を変更することはできません。以下の例では、
`example.com` は `192.0.2.0` に解決されます。

{{< text bash >}}
$ curl example.com -v

- Trying 192.0.2.0:80...
  {{< /text >}}

次に、このリクエストは Istio によってインターセプトされます。このとき、Istio はホスト名（
`Host: example.com` ヘッダーから）と宛先アドレス（`192.0.2.0:80`）を認識します。Istio は
これらの情報を使って、意図された宛先を決定します。
[トラフィックルーティングの理解](/ja/docs/ops/configuration/traffic-management/traffic-routing/)でこの動作の詳細を説明しています。

クライアントが DNS 解決に失敗した場合、Istio がリクエストを受け取る前に終了します。
つまり、Istio が認識しているホスト名（たとえば `ServiceEntry` で設定されたもの）であっても、
DNS で解決できなければリクエストは失敗します。
ただし、Istio の [DNS プロキシ](#dns-proxing)を使うことでこの動作を変更できます。

Istio が意図した宛先を決定した後、どのアドレスに送信するかを選択する必要があります。
Istio の高度な[負荷分散機能](/ja/docs/concepts/traffic-management/#load-balancing-options)により、
このアドレスはクライアントが送信した元の IP アドレスとは限りません。サービスの設定によって、Istio はいくつかの方法を取ります：

- クライアントの元の IP アドレスを使用（上記の例では `192.0.2.0`）。
  これは `resolution: NONE` タイプの `ServiceEntry` や[ヘッドレスサービス](https://kubernetes.io/ja/docs/concepts/services-networking/service/#headless-services)に該当します。
- 静的 IP アドレスのセットで負荷分散を行う。これは `resolution: STATIC` タイプの `ServiceEntry` で、`spec.endpoints` のすべてのアドレス、
  または標準の `Services` ではすべての `Endpoints` アドレスを使用します。
- DNS で定期的にアドレスを解決し、すべての結果で負荷分散を行う。これは
  `resolution: DNS` タイプの `ServiceEntry` に該当します。

いずれの場合も、Istio プロキシ内部の DNS 解決はユーザーアプリケーションの
DNS 解決とは独立しています。クライアントが DNS 解決を行っても、
プロキシは解決済みの IP アドレスを無視し、独自のアドレス（静的リストやプロキシ自身の DNS 解決結果）を使う場合があります。

## プロキシによる DNS 解決 {#proxying-dns-resolution}

多くのクライアントがリクエスト時に都度 DNS リクエストを行い（通常は結果をキャッシュ）、
Istio プロキシは同期的な DNS リクエストを一切行いません。`resolution: DNS`
タイプの `ServiceEntry` を設定すると、プロキシは設定されたホスト名を定期的（30 秒間隔、現時点で変更不可）に解決し、
すべてのリクエストにその結果を使用します。
この間隔は固定で、現在は変更できません。
プロキシがこれらのアプリケーションに一度もリクエストを送信していなくても、
この動作は発生します。

多数のプロキシや `resolution: DNS` タイプの `ServiceEntry` が多いメッシュでは、
特に TTL が短い場合、DNS サーバーへの負荷が高くなる可能性があります。
このような場合、以下の方法で負荷を軽減できます：

- `ServiceEntries` を `resolution: NONE` タイプに切り替えてプロキシの DNS ルックアップを完全に回避する。
  これは多くのユースケースで有効です。
- 解決対象のドメインを管理できる場合は、TTL を適切に長く設定する。
- `ServiceEntry` の対象ワークロードが少数の場合は、`exportTo` や [`Sidecar`](/ja/docs/reference/config/networking/sidecar/) でスコープを制限する。

## DNS プロキシ {#dns-proxing}

Istio は[DNS リクエストのプロキシ](/ja/docs/ops/configuration/traffic-management/dns-proxy/)機能を提供しています。
これにより、Istio はクライアントが送信した DNS リクエストをキャプチャし、直接応答を返すことができます。これにより DNS レイテンシが改善され、
負荷が軽減され、`ServiceEntries` が `kube-dns` で解決できない問題も解決できます。

このプロキシはユーザーアプリケーションが送信する DNS リクエストのみに適用されます。
`resolution: DNS` タイプの `ServiceEntries` でプロキシ自身が行う DNS 解決には影響しません。
