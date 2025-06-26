---
title: ztunnel は単一障害点（SPOF）ですか？
weight: 25
---

Istio の ztunnel は、Kubernetes クラスターに単一障害点（SPOF）を導入しません。
ztunnel の障害は、クラスター内の障害が発生しやすいコンポーネントとして扱われる単一のノードに限定されます。
その動作は、各クラスター上で実行される他のノードの重要なインフラストラクチャ（Linux カーネル、コンテナランタイムなど）と同じです。
合理的に設計されたシステムでは、ノードの中断はクラスターの中断を引き起こしません。[詳細はこちら](https://blog.howardjohn.info/posts/ambient-spof/)。
