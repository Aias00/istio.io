---
title: セキュリティ
description: Istio の認可と認証機能について説明します。
weight: 30
keywords:
  [
    security,
    policy,
    policies,
    authentication,
    authorization,
    rbac,
    access-control,
  ]
aliases:
  - /zh/docs/concepts/network-and-auth/auth.html
  - /zh/docs/concepts/security/authn-policy/
  - /zh/docs/concepts/security/mutual-tls/
  - /zh/docs/concepts/security/rbac/
  - /zh/docs/concepts/security/mutual-tls.html
  - /zh/docs/concepts/policies/
owner: istio/wg-security-maintainers
test: n/a
---

単一のアプリケーションをマイクロサービスに分割することで、柔軟性・スケーラビリティ・サービス再利用性など多くの利点が得られますが、マイクロサービスには特有のセキュリティ要件もあります：

- 中間者攻撃を防ぐためのトラフィック暗号化
- 柔軟なサービスアクセス制御のための双方向 TLS ときめ細かなアクセス制御ポリシー
- 誰がいつ何をしたかを特定するための監査ツール

Istio Security は、これらすべての課題に対応する包括的なセキュリティソリューションを提供します。
このページでは、どこでサービスを実行していても Istio のセキュリティ機能でサービスを保護できることを概観します。
特に、Istio セキュリティはデータ・エンドポイント・通信・プラットフォームに対する内外の脅威を軽減します。

{{< image width="75%"
    link="./overview.svg"
    caption="セキュリティ概要"
    >}}

Istio のセキュリティ機能は、強力なアイデンティティ、強力なポリシー、透過的な TLS 暗号化、認証/認可/監査（AAA）ツールを提供し、サービスとデータを保護します。Istio セキュリティの目標は：

- デフォルトで安全：アプリケーションコードやインフラの変更不要
- 多層防御：既存のセキュリティシステムと連携し多層防御を実現
- ゼロトラストネットワーク：信頼できないネットワーク上でも安全なソリューションを構築

[双方向 TLS 移行](/ja/docs/tasks/security/authentication/mtls-migration/)ガイドで、既存サービスに Istio セキュリティ機能を導入する方法を確認できます。[セキュリティタスク](/ja/docs/tasks/security/)もご覧ください。

## ハイレベルアーキテクチャ {#high-level-architecture}

Istio のセキュリティは複数のコンポーネントで構成されます：

- 鍵と証明書管理のための証明書認証局（CA）
- 設定 API サーバーがプロキシに配布する：
  - [認証ポリシー](/ja/docs/concepts/security/#authentication-policies)
  - [認可ポリシー](/ja/docs/concepts/security/#authorization-policies)
  - [セキュアネーミング情報](/ja/docs/concepts/security/#secure-naming)
- Sidecar およびエッジプロキシは[ポリシー実施ポイント](https://csrc.nist.gov/glossary/term/policy_enforcement_point)（PEP）として、クライアントとサーバー間通信のセキュリティを保護
- テレメトリーや監査を管理するための Envoy プロキシ拡張群

コントロールプレーンは API サーバーからの設定を処理し、データプレーンで PEP を設定します。
PEP は Envoy で実装されています。下図はアーキテクチャを示します。

{{< image width="75%"
    link="./arch-sec.svg"
    caption="セキュリティアーキテクチャ"
    >}}

以下で Istio セキュリティ機能の詳細を説明します。

## Istio アイデンティティ {#istio-identity}

アイデンティティはあらゆるセキュリティ基盤の基本概念です。ワークロード間通信の開始時、双方はアイデンティティ情報を含む証明書を交換し、双方向認証を行います。クライアント側では[セキュアネーミング](/ja/docs/concepts/security/#secure-naming)情報でサーバーのアイデンティティを検証し、そのワークロードが正当な実行者か確認します。サーバー側では[認可ポリシー](/ja/docs/concepts/security/#authorization-policies)に基づき、クライアントがアクセスできる情報を決定し、誰がいつ何にアクセスしたかを監査し、利用ワークロードに応じて課金し、未払いのクライアントのアクセスを拒否できます。

Istio のアイデンティティモデルは、リクエスト送信元のアイデンティティを特定する「サービスアイデンティティ」方式を採用しています。このモデルは柔軟かつ細粒度で、人間ユーザー・単一ワークロード・ワークロード群などをサービスアイデンティティで識別できます。サービスアイデンティティがないプラットフォームでは、Istio はサービス名などサービスインスタンスをグループ化できる他のアイデンティティも利用可能です。

各プラットフォームで利用できるサービスアイデンティティ例：

- Kubernetes：Kubernetes サービスアカウント
- GKE/GCE：GCP サービスアカウント
- オンプレミス（非 Kubernetes）：ユーザーアカウント、カスタムサービスアカウント、サービス名、Istio サービスアカウント、GCP サービスアカウント。カスタムサービスアカウントは既存サービスアカウントを参照し、顧客のアイデンティティディレクトリ管理のアイデンティティとして扱えます。

## アイデンティティと証明書管理 {#PKI}

Istio PKI は X.509 証明書を使い、各ワークロードに強力なアイデンティティを提供します。
`istio-agent` は各 Envoy プロキシとともに動作し、`istiod` と連携して大規模な鍵・証明書の自動ローテーションを実現します。下図はこの仕組みの流れを示します。

{{< tip >}}
訳注：ここで `istio-agent` という表現を使うのは、下図やその解説で "Istio agent" という用語が繰り返し使われているためです。実装上は、`istio-agent` は Sidecar コンテナ内の `pilot-agent` プロセスを指し、多機能ですが、ここでは特に Envoy との間で Unix socket 経由で SDS サービスをローカル提供している点が重要です。
{{< /tip >}}

{{< image width="75%"
    link="./id-prov.svg"
    caption="アイデンティティ供給フロー"
    >}}

Istio は以下の流れで鍵と証明書を提供します：

1. `istiod` は gRPC サービスで[証明書署名リクエスト](https://en.wikipedia.org/wiki/Certificate_signing_request)（CSR）を受け付けます。
1. `istio-agent` は起動時に秘密鍵と CSR を作成し、CSR と認証情報を `istiod` に送信して署名を依頼します。
1. `istiod` CA は CSR の認証情報を検証し、成功すれば CSR を署名して証明書を発行します。
1. ワークロード起動時、Envoy は [Secret Discovery Service（SDS）](https://www.envoyproxy.io/docs/envoy/latest/configuration/security/secret#secret-discovery-service-sds) API で同一コンテナ内の `istio-agent` に証明書・鍵を要求します。
1. `istio-agent` は `istiod` から受け取った証明書・鍵を Envoy の SDS API で渡します。
1. `istio-agent` はワークロード証明書の有効期限を監視し、上記プロセスを定期的に繰り返して証明書・鍵をローテーションします。

## 認証 {#authentication}

Istio は 2 種類の認証を提供します：

- ピア認証：サービス間認証で、接続元クライアントを検証します。
  Istio は[双方向 TLS](https://en.wikipedia.org/wiki/Mutual_authentication)をトランスポート認証のフルスタックソリューションとして提供し、サービスコードの変更なしで有効化できます。このソリューションは：

  - 各サービスに役割を表す強力なアイデンティティを提供し、クラスタやクラウドをまたいだ相互運用性を実現
  - サービス間通信のセキュリティを確保
  - 鍵管理システムで鍵・証明書の生成・配布・ローテーションを自動化

- リクエスト認証：エンドユーザー認証で、リクエストに付与された認証情報を検証します。
  Istio は JSON Web Token（JWT）検証でリクエストレベル認証を有効化し、カスタム認証や OpenID Connect 対応の認証（下記例など）を簡単に利用できます。
  - [ORY Hydra](https://www.ory.sh/)
  - [Keycloak](https://www.keycloak.org/)
  - [Auth0](https://auth0.com/)
  - [Firebase Auth](https://firebase.google.com/docs/auth/)
  - [Google Auth](https://developers.google.com/identity/protocols/OpenIDConnect)

いずれの場合も、Istio はカスタム Kubernetes API で認証ポリシーを `Istio config store` に保存します。
{{< gloss >}}Istiod{{< /gloss >}} は各プロキシを最新状態に保ち、必要に応じて鍵を提供します。また、Istio の認証機構は「パーミッシブモード（寛容モード）」をサポートし、強制適用前にポリシー変更がセキュリティ状況にどう影響するかを確認できます。

### 双方向 TLS 認証 {#mutual-TLS-authentication}

Istio はクライアント・サーバー両方の PEP でサービス間通信チャネルを確立します。
PEP は [Envoy プロキシ](https://www.envoyproxy.io/)で実装されています。
ワークロードが双方向 TLS 認証で他のワークロードにリクエストを送信する場合、処理は次の通りです：

1. Istio はクライアントのローカル Sidecar Envoy にクライアントからのアウトバウンドトラフィックをリダイレクトします。
1. クライアント Envoy とサーバー Envoy が双方向 TLS ハンドシェイクを開始します。ハンドシェイク中、クライアント Envoy は[セキュアネーミング](/ja/docs/concepts/security/#secure-naming)チェックも行い、サーバー証明書のサービスアカウントがターゲットサービスの実行を許可されているか検証します。
1. クライアント Envoy とサーバー Envoy が双方向 TLS 接続を確立し、Istio はクライアント Envoy からサーバー Envoy へトラフィックを転送します。
1. サーバー Envoy がリクエストを認可し、許可されればローカル TCP 接続でバックエンドサービスに転送します。

Istio はクライアント・サーバー両方に `TLSv1_2` を最低サポートバージョンとして、以下の暗号スイートを設定します：

- `ECDHE-ECDSA-AES256-GCM-SHA384`
- `ECDHE-RSA-AES256-GCM-SHA384`
- `ECDHE-ECDSA-AES128-GCM-SHA256`
- `ECDHE-RSA-AES128-GCM-SHA256`
- `AES256-GCM-SHA384`
- `AES128-GCM-SHA256`

#### パーミッシブモード {#permissive-mode}

Istio の双方向 TLS には「パーミッシブモード（寛容モード）」があり、サービスがプレーンテキストと双方向 TLS の両方のトラフィックを同時に受け入れられます。この機能により、双方向 TLS の導入が非常に容易になります。

運用者が双方向 TLS 有効化済みの Istio へサービスを移行する際、Istio Sidecar 未導入のクライアントやサーバーとの通信が問題になることがあります。多くの場合、すべてのクライアントに一度に Sidecar をインストールできず、そもそも権限がない場合もあります。サーバー側に Sidecar をインストールしても、既存接続を中断せずに双方向 TLS を有効化できません。

パーミッシブモードを有効にすると、サービスはプレーンテキストと双方向 TLS の両方を同時に受け入れます。このモードは導入時の柔軟性を大きく高めます。サーバーに Istio Sidecar をインストールすれば、既存のプレーンテキストトラフィックを中断せずに双方向 TLS トラフィックを即座に受け入れられます。運用者はクライアント側の Sidecar 導入・設定を段階的に進め、すべてのクライアントが対応したらサーバー側を TLS 専用モードに切り替えられます。詳細は[双方向 TLS 移行ガイド](/ja/docs/tasks/security/authentication/mtls-migration/)を参照してください。

#### セキュアネーミング {#secure-naming}

サーバーアイデンティティは証明書にエンコードされ、サービス名はサービスディスカバリや DNS で取得されます。セキュアネーミング情報はサーバーアイデンティティとサービス名をマッピングします。アイデンティティ `A` からサービス名 `B` へのマッピングは「`A` が `B` サービスの実行を許可されている」ことを意味します。コントロールプレーンは apiserver を監視し、セキュアネーミングマッピングを生成して PEP へ安全に配布します。以下はセキュアネーミングが認証で重要な理由の例です。

例えば、`datastore` サービスの正規サーバーは `infra-team` アイデンティティのみを使います。悪意あるユーザーが `test-team` アイデンティティの証明書・鍵を持っているとします。悪意あるユーザーは正規サービスを偽装し、クライアントから送信されるデータを盗み見ようとします。証明書と `test-team` アイデンティティの鍵で偽サーバーを立て、DNS スプーフィングや BGP/ルーティングハイジャック、ARP スプーフィングなどで `datastore` へのトラフィックを偽サーバーにリダイレクトします。

クライアントが `datastore` サービスを呼び出すと、サーバー証明書から `test-team` アイデンティティを抽出し、セキュアネーミング情報で `test-team` が `datastore` 実行を許可されているか検証します。`test-team` は許可されていないため認証は失敗します。

なお、非 HTTP/HTTPS トラフィックでは、DNS スプーフィング対策としてセキュアネーミングは機能しません。攻撃者が DNS を乗っ取り宛先 IP を変更した場合、TCP トラフィックにはホスト名情報が含まれないため、Envoy は宛先 IP だけでルーティングします。そのため、偽 IP のサービスにトラフィックが流れる可能性があります。この種の DNS スプーフィングはクライアント Envoy がトラフィックを受け取る前にも発生し得ます。

### 認証アーキテクチャ {#authentication-architecture}

ピア認証ポリシーとリクエスト認証ポリシーで、Istio メッシュ内でリクエストを受けるワークロードの認証要件を指定できます。運用者は `.yaml` ファイルでポリシーを記述し、デプロイ後は Istio 設定ストアに保存されます。Istio コントローラーは設定ストアを監視します。

ポリシーが変更されると、新しいポリシーは適切な設定に変換され、PEP に必要な認証メカニズムを指示します。コントロールプレーンは公開鍵を取得し、JWT 検証設定に付与できます。あるいは、Istiod が Istio 管理の鍵・証明書のパスを提供し、アプリケーション Pod にインストールして双方向 TLS に利用します。詳細は [PKI セクション](/ja/docs/concepts/security/#PKI) を参照してください。

Istio は非同期で設定をターゲットエンドポイントに送信します。プロキシが設定を受信すると、新しい認証要件が即時有効になります。

リクエスト送信側のクライアントサービスは、必要な認証メカニズムを遵守する責任があります。リクエスト認証では、アプリケーションが JWT 資格情報を取得しリクエストに付与します。ピア認証では、Istio が 2 つの PEP 間の全トラフィックを自動的に双方向 TLS にアップグレードします。認証ポリシーで双方向 TLS を無効化している場合、Istio は PEP 間でプレーンテキストを使い続けます。この動作を上書きするには、[DestinationRule](/ja/docs/concepts/traffic-management/#destination-rules) で双方向 TLS を明示的に無効化してください。双方向 TLS の詳細は[双方向 TLS 認証](/ja/docs/concepts/security/#mutual-TLS-authentication)を参照してください。

{{< image width="75%"
    link="./authn.svg"
    caption="認証アーキテクチャ"
    >}}

Istio はこの 2 種類の認証タイプと、資格情報内の他のクレーム（該当する場合）を次の層、すなわち[認可](/ja/docs/concepts/security/#authorization)へ出力します。

### 認証ポリシー {#authentication-policies}

このセクションでは Istio 認証ポリシーの詳細を説明します。
[認証アーキテクチャ](/ja/docs/concepts/security/#authentication-architecture)で述べたように、認証ポリシーはサービスが受信するリクエストに適用されます。双方向 TLS でクライアント認証ポリシーを指定するには、`DestinationRule` の `TLSSettings` を設定します。
[TLS 設定リファレンス](/ja/docs/reference/config/networking/destination-rule/#TLSSettings)も参照してください。

他の Istio 設定と同様、認証ポリシーは `.yaml` ファイルで記述し、`kubectl` で適用します。以下は `app: reviews` ラベルのワークロードへのトランスポート層認証に双方向 TLS を必須とする例です：

{{< text yaml >}}
apiVersion: security.istio.io/v1
kind: PeerAuthentication
metadata:
name: "example-peer-policy"
namespace: "foo"
spec:
selector:
matchLabels:
app: reviews
mtls:
mode: STRICT
{{< /text >}}

#### ポリシーストレージ {#policy-storage}

Istio はメッシュ全体のポリシーをルートネームスペースに保存します。これらのポリシーは空の selector でメッシュ内すべてのワークロードに適用されます。ネームスペース単位のポリシーは該当ネームスペースに保存され、そのネームスペース内のワークロードにのみ適用されます。`selector` フィールドを設定した場合、認証ポリシーは条件に一致するワークロードのみに適用されます。

ピア認証ポリシーとリクエスト認証ポリシーは kind フィールドで区別され、それぞれ `PeerAuthentication` と `RequestAuthentication` です。

#### selector フィールド {#selector-field}

ピア認証ポリシーとリクエスト認証ポリシーは `selector` フィールドで適用対象ワークロードのラベルを指定します。以下は `app: product-page` ラベルのワークロードに適用する selector フィールドの例です：

{{< text yaml >}}
selector:
matchLabels:
app: product-page
{{< /text >}}

`selector` フィールドを指定しない場合、Istio はそのポリシーをストレージ範囲内のすべてのワークロードに適用します。したがって、`selector` フィールドはポリシーの適用範囲を指定するのに役立ちます：

- メッシュ全体ポリシー：ルートネームスペースで selector なしまたは空で指定
- ネームスペース単位ポリシー：非ルートネームスペースで selector なしまたは空で指定
- ワークロード単位ポリシー：通常のネームスペースで selector ありで指定

ピア認証ポリシーとリクエスト認証ポリシーは selector フィールドの階層原則に従いますが、Istio はこれらをやや異なる方法で組み合わせ・適用します。

メッシュ全体のピア認証ポリシーは 1 つだけ、各ネームスペース単位のピア認証ポリシーも 1 つだけです。同じ範囲で複数のピア認証ポリシーを設定した場合、Istio は新しい方を無視します。複数のワークロード単位ピア認証ポリシーが一致した場合、Istio は最も古いものを選択します。

Istio は各ワークロードに対し、最も狭い範囲の一致ポリシーを次の順で適用します：

1. ワークロード単位
1. ネームスペース単位
1. メッシュ全体

Istio はすべての一致するリクエスト認証ポリシーを組み合わせ、1 つのリクエスト認証ポリシーとして扱います。したがって、メッシュやネームスペース単位で複数のリクエスト認証ポリシーを設定できますが、複数のメッシュ全体・ネームスペース単位リクエスト認証ポリシーは避けるのがベストプラクティスです。

#### ピア認証 {#peer-authentication}

ピア認証ポリシーは、Istio がターゲットワークロードに適用する双方向 TLS モードを指定します。サポートされるモード：

- PERMISSIVE：ワークロードは双方向 TLS とプレーンテキストの両方を受け入れます。
  Sidecar 未導入ワークロードの移行時に便利です。移行完了後は STRICT へ切り替えます。
- STRICT：ワークロードは双方向 TLS のみ受け入れます。
- DISABLE：双方向 TLS を無効化します。独自のセキュリティ対策がない限り、このモードは推奨しません。

モードが unset の場合、親スコープのモードを継承します。unset のメッシュ全体ピア認証ポリシーはデフォルトで `PERMISSIVE` です。

以下はネームスペース `foo` のすべてのワークロードに双方向 TLS を必須とするピア認証ポリシー例です：

{{< text yaml >}}
apiVersion: security.istio.io/v1
kind: PeerAuthentication
metadata:
name: "example-policy"
namespace: "foo"
spec:
mtls:
mode: STRICT
{{< /text >}}

ワークロード単位のピア認証ポリシーでは、異なるポートごとに異なる双方向 TLS モードを指定できます。ポート単位の設定はワークロードが宣言したポートのみで有効です。以下は `app:example-app` ワークロードの 80 番ポートで双方向 TLS を無効化し、他のポートはネームスペース単位の設定を継承する例です：

{{< text yaml >}}
apiVersion: security.istio.io/v1
kind: PeerAuthentication
metadata:
name: "example-workload-policy"
namespace: "foo"
spec:
selector:
matchLabels:
app: example-app
portLevelMtls:
80:
mode: DISABLE
{{< /text >}}

上記ピア認証ポリシーは、以下のような Service 定義がある場合に有効です。
`example-service` へのリクエストが `example-app` ワークロードの 80 番ポートにバインドされます。

{{< text yaml >}}
apiVersion: v1
kind: Service
metadata:
name: example-service
namespace: foo
spec:
ports:

- name: http
  port: 8000
  protocol: TCP
  targetPort: 80
  selector:
  app: example-app
  {{< /text >}}

#### リクエスト認証 {#request-authentication}

リクエスト認証ポリシーは JSON Web Token（JWT）検証に必要な値を指定します。これには：

- トークンのリクエスト内での位置
- リクエストの issuer
- 公開 JSON Web Key Set（JWKS）

Istio はリクエスト認証ポリシーのルールに従い、提供されたトークン（あれば）を検証し、無効なトークンのリクエストを拒否します。トークンがない場合、デフォルトでリクエストは許可されます。トークンなしリクエストを拒否したい場合は、特定の操作（パスやメソッドなど）を制限する認可ルールを追加してください。

リクエスト認証ポリシーで一意の位置を使う場合、複数の JWT を指定できます。複数のポリシーが 1 つのワークロードに一致した場合、Istio はすべてのルールを 1 つのポリシーとして組み合わせます。これは異なる JWT プロバイダーを受け入れるワークロードの開発に便利です。ただし、複数の有効な JWT を持つリクエストはサポートされません（出力主体が未定義のため）。

#### Principal {#principals}

ピア認証ポリシーと双方向 TLS を使う場合、Istio はピア認証からアイデンティティを `source.principal` に抽出します。同様に、リクエスト認証ポリシーでは JWT のアイデンティティを `request.auth.principal` に設定します。これらの principal を使って認可ポリシーを設定したり、テレメトリー出力に利用できます。

### 認証ポリシーの更新 {#updating-authentication-policies}

認証ポリシーはいつでも変更でき、Istio はほぼリアルタイムで新しいポリシーをワークロードにプッシュします。ただし、すべてのワークロードが同時に新ポリシーを受信する保証はありません。以下の推奨事項で認証ポリシー更新時の混乱を防げます：

- ピア認証ポリシーのモードを `DISABLE` から `STRICT` に変更する場合、`PERMISSIVE` モードを経由してください。逆も同様です。すべてのワークロードが切り替わったら最終モードにします。Istio テレメトリーで切り替え状況を確認できます。
- リクエスト認証ポリシーをある JWT から別の JWT へ移行する場合、新 JWT のルールを追加し、旧ルールは削除しないでください。すべてのトラフィックが新 JWT に切り替わったら旧ルールを削除します。ただし、各 JWT は異なる位置を使う必要があります。

## 認可 {#authorization}

Istio の認可機能は、メッシュ内ワークロードに対しメッシュ・ネームスペース・ワークロード単位のアクセス制御を提供します。この階層的制御により：

- ワークロード間・エンドユーザーからワークロードへの認可
- シンプルな API：単一で使いやすく保守しやすい [`AuthorizationPolicy` CRD](/ja/docs/reference/config/security/authorization-policy/)
- 柔軟なセマンティクス：Istio 属性でカスタム条件を定義し、DENY/ALLOW アクションを利用可能
- 高パフォーマンス：認可は Envoy 上でローカルに強制
- 高い互換性：HTTP/HTTPS/HTTP2 や任意の TCP プロトコルをネイティブサポート

### 認可アーキテクチャ {#authorization-architecture}

認可ポリシーはサーバー側 Envoy プロキシのインバウンドトラフィックにアクセス制御を適用します。
各 Envoy プロキシは認可エンジンを実行し、リクエスト到着時に現在の認可ポリシーでリクエストコンテキストを評価し、`ALLOW` または `DENY` を返します。運用者は `.yaml` ファイルで Istio 認可ポリシーを指定します。

{{< image width="50%"
    link="./authz.svg"
    caption="認可アーキテクチャ"
    >}}

### 暗黙的有効化 {#implicit-enablement}

Istio の認可機能は明示的な有効化不要で、インストール後すぐ利用できます。ワークロードにアクセス制御を適用するには認可ポリシーを適用してください。

認可ポリシーが適用されていないワークロードには、Istio はすべてのリクエストを許可します。

認可ポリシーは `ALLOW`、`DENY`、`CUSTOM` アクションをサポートします。必要に応じて複数のポリシーを適用し、各ポリシーで異なるアクションを指定してワークロードへのアクセスを安全に制御できます。

Istio は `CUSTOM`、`DENY`、`ALLOW` の順で各レイヤーの一致ポリシーをチェックします。各アクションタイプで、まずアクションが適用されたポリシーがあるかを確認し、次にリクエストがポリシールールに一致するかを確認します。いずれのレイヤーでも一致しなければ、次のレイヤーに進みます。

下図はポリシー優先順位の詳細です：

{{< image width="50%" link="./authz-eval.svg" caption="認可ポリシー優先順位">}}

同じワークロードに複数の認可ポリシーを適用した場合、Istio はそれらを累積的に適用します。

### 認可ポリシー {#authorization-policies}

認可ポリシーを設定するには、[`AuthorizationPolicy` カスタムリソース](/ja/docs/reference/config/security/authorization-policy/)を作成します。認可ポリシーは selector（選択子）、action（アクション）、rules（ルール）リストで構成されます：

- `selector` フィールド：ポリシーの対象を指定
- `action` フィールド：リクエストを許可するか拒否するか指定
- `rules`：アクションをトリガーする条件を指定
  - `rules` の `from` フィールド：リクエストの送信元を指定
  - `rules` の `to` フィールド：リクエストの操作を指定
  - `rules` の `when` フィールド：ルール適用に必要な条件を指定

以下は、2 つの送信元（サービスアカウント `cluster.local/ns/default/sa/curl` とネームスペース `dev`）が有効な JWT トークンでリクエストした場合、ネームスペース `foo` の `app: httpbin`・`version: v1` ラベル付きワークロードへのアクセスを許可する認可ポリシー例です。

{{< text yaml >}}
apiVersion: security.istio.io/v1
kind: AuthorizationPolicy
metadata:
name: httpbin
namespace: foo
spec:
selector:
matchLabels:
app: httpbin
version: v1
action: ALLOW
rules:

- from:
  - source:
    principals: ["cluster.local/ns/default/sa/curl"]
  - source:
    namespaces: ["dev"]
    to:
  - operation:
    methods: ["GET"]
    when:
  - key: request.auth.claims[iss]
    values: ["https://accounts.google.com"]
    {{< /text >}}

次は、送信元がネームスペース `foo` でない場合にリクエストを拒否する認可ポリシー例です。

{{< text yaml >}}
apiVersion: security.istio.io/v1
kind: AuthorizationPolicy
metadata:
name: httpbin-deny
namespace: foo
spec:
selector:
matchLabels:
app: httpbin
version: v1
action: DENY
rules:

- from:
  - source:
    notNamespaces: ["foo"]
    {{< /text >}}

DENY ポリシーは ALLOW ポリシーより優先されます。リクエストが ALLOW と DENY の両方に一致した場合、リクエストは拒否されます。Istio はまず DENY ポリシーを評価し、ALLOW ポリシーで DENY を回避できないようにします。

#### ポリシー対象 {#policy-target}

`metadata/namespace` フィールドとオプションの `selector` フィールドでポリシーの範囲や対象を指定できます。`metadata/namespace` でポリシーの適用ネームスペースを指定します。ルートネームスペースに設定すると、メッシュ内すべてのネームスペースに適用されます。ルートネームスペースのデフォルトは `istio-system` です。他のネームスペースを指定した場合、そのネームスペース内のみに適用されます。

`selector` フィールドでさらに対象を絞り、特定ワークロードのみに適用できます。`selector` はラベルの `{key: value}` リストです。未設定の場合、認可ポリシーは同じネームスペース内のすべてのワークロードに適用されます。

以下の `allow-read` ポリシーは、`default` ネームスペースの `app: products` ラベル付きワークロードへの "GET"・"HEAD" 操作を許可します。

{{< text yaml >}}
apiVersion: security.istio.io/v1
kind: AuthorizationPolicy
metadata:
name: allow-read
namespace: default
spec:
selector:
matchLabels:
app: products
action: ALLOW
rules:

- to: - operation:
  methods: ["GET", "HEAD"]
  {{< /text >}}

#### 値のマッチング {#value-matching}

認可ポリシーのほとんどのフィールドは、以下のすべてのマッチングパターンをサポートします：

- 完全一致：完全な文字列一致
- プレフィックス一致：末尾が `*` の文字列（例：`test.abc.*` は `test.abc.com` などに一致）
- サフィックス一致：先頭が `*` の文字列（例：`*.abc.com` は `eng.abc.com` などに一致）
- 存在一致：`*` で任意の非空値を指定（例：`fieldname: ["*"]` で必須フィールドを指定）。未指定は空も含めて一致。

例外もあります。以下のフィールドは完全一致のみサポート：

- `when` 部分の `key` フィールド
- `source` 部分の `ipBlocks`
- `to` 部分の `ports` フィールド

以下の例は `/test/*` プレフィックスまたは `*/info` サフィックスのパスへのアクセスを許可するポリシーです。

{{< text yaml >}}
apiVersion: security.istio.io/v1
kind: AuthorizationPolicy
metadata:
name: tester
namespace: default
spec:
selector:
matchLabels:
app: products
action: ALLOW
rules:

- to: - operation:
  paths: ["/test/*", "*/info"]
  {{< /text >}}

#### 除外マッチング {#exclusion-matching}

`when` の `notValues`、`source` の `notIpBlocks`、`to` の `notPorts` など、否定条件のマッチングもサポートしています。

以下は `/healthz` 以外のパスでは JWT 認証主体が必須となる例です。これにより `/healthz` へのリクエストは JWT 認証から除外されます：

{{< text yaml >}}
apiVersion: security.istio.io/v1
kind: AuthorizationPolicy
metadata:
name: disable-jwt-for-healthz
namespace: default
spec:
selector:
matchLabels:
app: products
action: ALLOW
rules:

- to: - operation:
  notPaths: ["/healthz"]
  from: - source:
  requestPrincipals: ["*"]
  {{< /text >}}

次は `/admin` パスでリクエスト主体がない場合に拒否する例です：

{{< text yaml >}}
apiVersion: security.istio.io/v1
kind: AuthorizationPolicy
metadata:
name: enable-jwt-for-admin
namespace: default
spec:
selector:
matchLabels:
app: products
action: DENY
rules:

- to: - operation:
  paths: ["/admin"]
  from: - source:
  notRequestPrincipals: ["*"]
  {{< /text >}}

#### `allow-nothing`、`deny-all`、`allow-all` ポリシー {#allow-nothing-deny-all-and-allow-all-policy}

以下は何も一致しない `ALLOW` ポリシー例です。他に `ALLOW` ポリシーがなければ、リクエストは「デフォルト拒否」動作で常に拒否されます。

{{< tip >}}
`allow-nothing` ポリシーから始めて、徐々に `ALLOW` ポリシーを追加してアクセスを開放していくのが安全な運用です。
{{< /tip >}}

{{< text yaml >}}
apiVersion: security.istio.io/v1
kind: AuthorizationPolicy
metadata:
name: allow-nothing
spec:
action: ALLOW

# rules フィールド未指定で常に一致しない

{{< /text >}}

以下は明示的にすべてのアクセスを拒否する `DENY` ポリシー例です。他に `ALLOW` ポリシーがあっても、`DENY` が優先されるため常に拒否されます。一時的にすべてのアクセスを遮断したい場合に使えます。

{{< text yaml >}}
apiVersion: security.istio.io/v1
kind: AuthorizationPolicy
metadata:
name: deny-all
spec:
action: DENY

# rules フィールドに空白ルールで常に一致

rules:

- {}
  {{< /text >}}

以下はワークロードへの完全アクセスを許可する `ALLOW` ポリシー例です。
それにより他の `ALLOW` ポリシーは無効化されます。一時的に完全公開したい場合に使えます。ただし、`CUSTOM` や `DENY` ポリシーで拒否される場合もあります。

{{< text yaml >}}
apiVersion: security.istio.io/v1
kind: AuthorizationPolicy
metadata:
name: allow-all
spec:
action: ALLOW

# これにより他の ALLOW ポリシーは無効化されます

rules:

- {}
  {{< /text >}}

#### カスタム条件 {#custom-conditions}

`when` セクションで追加条件も指定できます。以下の `AuthorizationPolicy` 例では、`request.headers [version]` が `v1` または `v2` の場合のみ許可します。ここで key は Istio 属性 `request.headers`（辞書型）の 1 項目です。

{{< text yaml >}}
apiVersion: security.istio.io/v1
kind: AuthorizationPolicy
metadata:
name: httpbin
namespace: foo
spec:
selector:
matchLabels:
app: httpbin
version: v1
action: ALLOW
rules:

- from:
  - source:
    principals: ["cluster.local/ns/default/sa/curl"]
    to:
  - operation:
    methods: ["GET"]
    when:
  - key: request.headers[version]
    values: ["v1", "v2"]
    {{< /text >}}

[条件ページ](/ja/docs/reference/config/security/conditions/)でサポートされる条件 key 値を確認できます。

#### 認証済み・未認証アイデンティティ {#authenticated-and-unauthenticated-identity}

ワークロードを公開したい場合、`source` セクションを空にします。これで認証済み・未認証のすべてのユーザー・ワークロードからのリクエストが許可されます：

{{< text yaml >}}
apiVersion: security.istio.io/v1
kind: AuthorizationPolicy
metadata:
name: httpbin
namespace: foo
spec:
selector:
matchLabels:
app: httpbin
version: v1
action: ALLOW
rules:

- to:
  - operation:
    methods: ["GET", "POST"]
    {{< /text >}}

認証済みユーザーのみ許可したい場合は `principal` を `"*"` に設定します：

{{< text yaml >}}
apiVersion: security.istio.io/v1
kind: AuthorizationPolicy
metadata:
name: httpbin
namespace: foo
spec:
selector:
matchLabels:
app: httpbin
version: v1
action: ALLOW
rules:

- from:
  - source:
    principals: ["*"]
    to:
  - operation:
    methods: ["GET", "POST"]
    {{< /text >}}

### プレーン TCP プロトコルでの Istio 認可 {#using-Istio-authorization-on-plain-TCP-protocols}

Istio 認可は MongoDB など任意のプレーン TCP プロトコルを使うワークロードもサポートします。この場合、HTTP ワークロードと同様に認可ポリシーを設定できますが、一部のフィールドや条件は HTTP ワークロード専用です。これには：

- 認可ポリシー `source` の `request_principals`
- 認可ポリシー `operation` の `hosts`、`methods`、`paths`

[条件ページ](/ja/docs/reference/config/security/conditions/)でサポートされる条件を確認できます。TCP ワークロードで HTTP 専用フィールドを使った場合、Istio はそれらを無視します。

たとえば、ポート `27017` で MongoDB サービスがある場合、以下の認可ポリシーで Istio メッシュ内の `bookinfo-ratings-v2` サービスだけがアクセス可能になります：

{{< text yaml >}}
apiVersion: security.istio.io/v1
kind: AuthorizationPolicy
metadata:
name: mongodb-policy
namespace: default
spec:
selector:
matchLabels:
app: mongodb
action: ALLOW
rules:

- from:
  - source:
    principals: ["cluster.local/ns/default/sa/bookinfo-ratings-v2"]
    to:
  - operation:
    ports: ["27017"]
    {{< /text >}}

### 双方向 TLS への依存 {#dependency-on-mutual-TLS}

Istio は双方向 TLS でクライアントからサーバーへ安全に情報を伝達します。認可ポリシーで以下のフィールドを使う場合は、まず双方向 TLS を有効化してください：

- `source` の `principals`・`notPrincipals`
- `source` の `namespaces`・`notNamespaces`
- `source.principal` カスタム条件
- `source.namespace` カスタム条件

これらのフィールドは、`PeerAuthentication` で `STRICT` 双方向 TLS モードを使うことを強く推奨します。`PERMISSIVE` モードでプレーンテキストトラフィックが混在すると、意図しないリクエスト拒否やセキュリティポリシー回避が発生する可能性があります。

STRICT モードが使えない場合は、[セキュリティアドバイザリ](/ja/news/security/istio-security-2021-004)で詳細や代替策を確認してください。

## さらに学ぶ {#learn-more}

上記の基本概念を学んだ後は、以下も参考にしてください：

- [認証](/ja/docs/tasks/security/authentication)・[認可](/ja/docs/tasks/security/authorization)タスクでセキュリティポリシーを試す
- セキュリティ強化に役立つ[ポリシー例](/ja/docs/ops/configuration/security/security-policy-examples)を確認
- [FAQ](/ja/docs/ops/common-problems/security-issues/)でトラブル時の解決策を学ぶ
