---
title: Ambient と Istio コントロールプレーン
description: Ambient が Istio コントロールプレーンとどのようにやり取りするかを理解します。
weight: 2
owner: istio/wg-networking-maintainers
test: no
---

すべての Istio {{< gloss "data plane" >}}データプレーン{{< /gloss >}}モードと同様に、
Ambient は Istio {{< gloss "control plane" >}}コントロールプレーン{{< /gloss>}}を使用します。
Ambient では、コントロールプレーンは各 Kubernetes ノード上の {{< gloss >}}ztunnel{{< /gloss >}} プロキシと通信します。

この図は、ztunnel プロキシと `istiod` コントロールプレーン、およびコントロールプレーン関連コンポーネントの概要を示しています。

{{< image width="100%"
link="ztunnel-architecture.svg"
caption="Ztunnel 架构"
>}}

ztunnel プロキシは xDS API を使用して Istio コントロールプレーン（`istiod`）と通信します。
これにより、現代の分散システムで必要な高速で動的な設定更新が可能になります。
ztunnel プロキシは、xDS を使用して Kubernetes ノード上でスケジュールされたすべての Pod の Service Account に対して、
{{< gloss "mutual tls authentication" >}}mTLS{{< /gloss >}} 証明書を取得します。
単一の ztunnel プロキシは、L4 
データプレーン機能を実現するために、共有されているノード上の任意の Pod を代表することができます。
これには、関連する設定と証明書を取得する必要があります。
この多租戦構成は、Sidecar モードとは対照的に、鮮明な違いがあります。

また、Ambient モードでは、xDS API で一連の簡略化されたリソースを使用して ztunnel プロキシの設定を行うことに注意してください。
これにより、パフォーマンスが向上し（istiod から ztunnel プロキシに送信される情報セットが小さくなるため）、トラブルシューティングが容易になります。
