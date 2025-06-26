---
title: MySQL 接続のトラブルシューティング
description: PERMISSIVE モードによる MySQL 接続問題の解決。
weight: 95
keywords: [mysql,mtls]
---

Istio をインストールすると、MySQL が接続できない場合があります。
これは、MySQL が[サーバー優先](/ja/docs/ops/deployment/application-requirements/#server-first-protocols)プロトコルであるためです。
これは、Istio のプロトコル検出を妨げます。
特に、`PERMISSIVE` mTLS モードを使用すると、問題が発生する可能性があります。
`ERROR 2013 (HY000): Lost connection to MySQL server at
'reading initial communication packet', system error: 0`
のようなエラーが表示される場合があります。

これは、`STRICT` または `DISABLE` モードを使用するか、すべてのクライアントが mTLS を送信するように構成することで解決できます。
[サーバー優先プロトコル](/ja/docs/ops/deployment/application-requirements/#server-first-protocols)を参照してください。
