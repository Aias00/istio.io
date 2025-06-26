---
title: ztunnel トラフィックリダイレクト
description: トラフィックが Pod と ztunnel ノードプロキシの間でどのようにリダイレクトされるかを理解します。
weight: 2
aliases:
  - /zh/docs/ops/ambient/usage/traffic-redirection
  - /zh/latest/docs/ops/ambient/usage/traffic-redirection
owner: istio/wg-networking-maintainers
test: no
---

Ambient モードのコンテキストでは、**トラフィックリダイレクト**はデータプレーン機能を指します。
この機能は、Ambient モードで動作するワークロードに送信されるトラフィックを攔截し、
{{< gloss >}}ztunnel{{< /gloss >}} ノードプロキシを介してそれらをルーティングします。
有時は**トラフィックキャプチャ**という用語も使用されます。

ztunnel はアプリケーショントラフィックを透過的に暗号化およびルーティングすることを目的としているため、
「メッシュ内」の Pod に入って出るすべてのトラフィックをキャプチャする必要があります。
これは安全に重要なタスクです：ztunnel をバイパスできれば、認証ポリシーをバイパスできます。

## Istio の in-Pod トラフィックリダイレクトモデル {#istio-s-in-pod-traffic-redirection-model}

Ambient モードの in-Pod トラフィックリダイレクトの核心設計原則は、
ztunnel プロキシがワークロード Pod の Linux ネットワーク名前空間内でデータパスキャプチャを実行できることです。
これは [`istio-cni` ノードプロキシ](/zh/docs/setup/additional-setup/cni/)と
ztunnel ノードプロキシの機能協調によって実現されます。
このモデルの主な利点の 1 つは、Istio の Ambient モードが任意の Kubernetes CNI プラグインと透明に連携できることです。
Kubernetes ネットワーク機能に影響を与えることなく。

以下の図は、Ambient モードで動作するワークロード Pod が起動（または追加）されたときのイベントの順序を示しています。

{{< image width="100%"
    link="./pod-added-to-ambient.svg"
    alt="Pod が Ambient メッシュに追加される"
    >}}

`istio-cni` ノードプロキシは、Pod の作成と削除などの CNI イベントに応答し、
また、Ambient ラベルが追加された Pod または名前空間などの Kubernetes API サーバーのイベントを監視します。

`istio-cni` ノードプロキシは、Kubernetes クラスター内の主流 CNI プラグインの後にコンテナランタイムによって実行されるチェーン CNI プラグインをインストールします。
その唯一の目的は、コンテナランタイムが既に Ambient モードで登録された名前空間内に新しい Pod を作成するときに `istio-cni` ノードプロキシに通知し、
新しい Pod のコンテキストを `istio-cni` に伝播することです。

`istio-cni` ノードプロキシが Pod をメッシュに追加する必要があることを通知すると、
以下の一連の操作が実行されます：

- `istio-cni` は Pod のネットワーク名前空間に入り、ネットワークリダイレクトルールを確立して、
  データパケットを攔截し、[既知のポート](https://github.com/istio/ztunnel/blob/master/ARCHITECTURE.md#ports)
  （15008, 15006, 15001）でリスニングするノードローカル ztunnel プロキシインスタンスに透過的にリダイレクトします。

- その後、`istio-cni` ノードプロキシは Unix ドメインソケットを使用して ztunnel プロキシに通知し、
  Pod のネットワーク名前空間内にローカルプロキシリスニングポート（ポート 15008、15006、および 15001 上）を確立し、
  ztunnel に Pod ネットワーク名前空間を表す低レベルの Linux [ファイルディスクリプター](https://en.wikipedia.org/wiki/File_descriptor)を提供します。
    - 通常、ソケットは実際にそのネットワーク名前空間内で実行されるプロセスによって Linux ネットワーク名前空間内で作成されますが、
      低レベルのソ

- ノードローカル ztunnel は内部で新しい論理プロキシインスタンスとリスニングポートセットを起動し、
  新しく追加された Pod 専用です。注意：これはまだ同じプロセス内で実行されており、
  単に Pod 専用のタスクです。

- in-Pod リダイレクトルールが確立され、ztunnel がリスニングポートを確立すると、
  Pod がメッシュに追加され、トラフィックがノードローカル ztunnel を介して流れ始めます。

デフォルトでは、メッシュ内の Pod 間のトラフィックは mTLS で完全に暗号化されます。

現在、データは Pod ネットワーク名前空間に入って出ると暗号化されます。
メッシュ内の各 Pod はメッシュポリシーを実行し、トラフィックを安全に暗号化できます。
Pod 内で実行されるユーザーアプリケーションがそれを知らなくても。

以下の図は、新しいモデルで Ambient メッシュ内の Pod 間の暗号化トラフィックの流れを示しています：

{{< image width="100%"
    link="./traffic-flows-between-pods-in-ambient.svg"
    alt="HBONE トラフィックが Ambient メッシュ内の Pod 間で流れる"
    >}}

## Ambient モードでのトラフィックリダイレクトの観察とデバッグ {#observing-and-debugging-traffic-redirection-in-ambient-mode}

Ambient モードでトラフィックリダイレクトが正常に動作しない場合、問題の範囲を狭めるためにいくつかの迅速なチェックを実行できます。
[ztunnel デバッグガイド](/zh/docs/ambient/usage/troubleshoot-ztunnel/)で説明されている手順から始めることをお勧めします。

### ztunnel プロキシのログを確認 {#check-the-ztunnel-proxy-logs}

アプリケーション Pod が Ambient メッシュの一部である場合、
ztunnel プロキシのログを確認して、メッシュがトラフィックをリダイレクトしていることを確認できます。
以下の例では、`inpod` に関連する ztunnel ログは、Pod 内リダイレクトモードが有効になっていることを示しています。
プロキシは、Ambient アプリケーション Pod のネットワーク名前空間（netns）情報を受信し、
そのプロキシを開始しました。

{{< text bash >}}
$ kubectl logs ds/ztunnel -n istio-system  | grep inpod
Found 3 pods, using pod/ztunnel-hl94n
inpod_enabled: true
inpod_uds: /var/run/ztunnel/ztunnel.sock
inpod_port_reuse: true
inpod_mark: 1337
2024-02-21T22:01:49.916037Z  INFO ztunnel::inpod::workloadmanager: handling new stream
2024-02-21T22:01:49.919944Z  INFO ztunnel::inpod::statemanager: pod WorkloadUid("1e054806-e667-4109-a5af-08b3e6ba0c42") received netns, starting proxy
2024-02-21T22:01:49.925997Z  INFO ztunnel::inpod::statemanager: pod received snapshot sent
2024-02-21T22:03:49.074281Z  INFO ztunnel::inpod::statemanager: pod delete request, draining proxy
2024-02-21T22:04:58.446444Z  INFO ztunnel::inpod::statemanager: pod WorkloadUid("1e054806-e667-4109-a5af-08b3e6ba0c42") received netns, starting proxy
{{< /text >}}

### ソケットの状態を確認 {#confirm-the-state-of-sockets}

以下の手順に従って、ポート 15001、15006、および 15008 上のソケットが開いてリスニング状態になっていることを確認してください。

{{< text bash >}}
$ kubectl debug $(kubectl get pod -l app=curl -n ambient-demo -o jsonpath='{.items[0].metadata.name}') -it -n ambient-demo  --image nicolaka/netshoot  -- ss -ntlp
Defaulting debug container name to debugger-nhd4d.
State  Recv-Q Send-Q Local Address:Port  Peer Address:PortProcess
LISTEN 0      128        127.0.0.1:15080      0.0.0.0:*
LISTEN 0      128                *:15006            *:*
LISTEN 0      128                *:15001            *:*
LISTEN 0      128                *:15008            *:*
{{< /text >}}

### iptables ルールの設定を確認 {#check-the-iptables-rules-setup}

アプリケーション内の 1 つの Pod の iptables ルールの設定を確認するには、以下のコマンドを実行してください：

{{< text bash >}}
$ kubectl debug $(kubectl get pod -l app=curl -n ambient-demo -o jsonpath='{.items[0].metadata.name}') -it --image gcr.io/istio-release/base --profile=netadmin -n ambient-demo -- iptables-save

Defaulting debug container name to debugger-m44qc.
# iptables-save によって生成された
*mangle
:PREROUTING ACCEPT [320:53261]
:INPUT ACCEPT [23753:267657744]
:FORWARD ACCEPT [0:0]
:OUTPUT ACCEPT [23352:134432712]
:POSTROUTING ACCEPT [23352:134432712]
:ISTIO_OUTPUT - [0:0]
:ISTIO_PRERT - [0:0]
-A PREROUTING -j ISTIO_PRERT
-A OUTPUT -j ISTIO_OUTPUT
-A ISTIO_OUTPUT -m connmark --mark 0x111/0xfff -j CONNMARK --restore-mark --nfmask 0xffffffff --ctmask 0xffffffff
-A ISTIO_PRERT -m mark --mark 0x539/0xfff -j CONNMARK --set-xmark 0x111/0xfff
-A ISTIO_PRERT -s 169.254.7.127/32 -p tcp -m tcp -j ACCEPT
-A ISTIO_PRERT ! -d 127.0.0.1/32 -i lo -p tcp -j ACCEPT
-A ISTIO_PRERT -p tcp -m tcp --dport 15008 -m mark ! --mark 0x539/0xfff -j TPROXY --on-port 15008 --on-ip 0.0.0.0 --tproxy-mark 0x111/0xfff
-A ISTIO_PRERT -p tcp -m conntrack --ctstate RELATED,ESTABLISHED -j ACCEPT
-A ISTIO_PRERT ! -d 127.0.0.1/32 -p tcp -m mark ! --mark 0x539/0xfff -j TPROXY --on-port 15006 --on-ip 0.0.0.0 --tproxy-mark 0x111/0xfff
COMMIT
# 完了
# iptables-save によって生成された
*nat
:PREROUTING ACCEPT [0:0]
:INPUT ACCEPT [0:0]
:OUTPUT ACCEPT [175:13694]
:POSTROUTING ACCEPT [205:15494]
:ISTIO_OUTPUT - [0:0]
-A OUTPUT -j ISTIO_OUTPUT
-A ISTIO_OUTPUT -d 169.254.7.127/32 -p tcp -m tcp -j ACCEPT
-A ISTIO_OUTPUT -p tcp -m mark --mark 0x111/0xfff -j ACCEPT
-A ISTIO_OUTPUT ! -d 127.0.0.1/32 -o lo -j ACCEPT
-A ISTIO_OUTPUT ! -d 127.0.0.1/32 -p tcp -m mark ! --mark 0x539/0xfff -j REDIRECT --to-ports 15001
COMMIT
{{< /text >}}

コマンドの出力によると、追加の Istio 固有のチェーンがアプリケーション Pod のネットワーク名前空間内の netfilter/iptables の NAT テーブルと Mangle テーブルに追加されています。
すべての Pod に入る TCP トラフィックは、入力処理のために ztunnel プロキシにリダイレクトされます。
もしトラフィックが平文（送信元ポートが 15008 でない）であれば、in-Pod ztunnel の平文リスニングポート 15006 にリダイレクトされます。
もしトラフィックが HBONE（送信元ポートが 15008）であれば、in-Pod ztunnel の HBONE リスニングポート 15008 にリダイレクトされます。
すべての Pod から出る TCP トラフィックは、出口処理のために ztunnel のポート 15001 にリダイレクトされ、
