---
---
`{{< istio_previous_version_revision >}}-1` と `{{< istio_full_version_revision >}}` の 2 つのリビジョンをインストールしたクラスターを考えてみます。
クラスター管理者は、古い `{{< istio_previous_version_revision >}}-1` リビジョンを指す `prod-stable` リビジョンラベルを作成し、
新しい `{{< istio_full_version_revision >}}` リビジョンを指す `prod-canary` リビジョンラベルを作成します。
以下のコマンドを実行することで、この状態になります：
