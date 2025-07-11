---
title: クリーンアップ
description: ローカリティ負荷分散のクリーンアップ手順。
weight: 30
icon: tasks
keywords: [locality, load balancing]
test: yes
owner: istio/wg-networking-maintainers
---

ローカリティ負荷分散タスクが完了したので、クリーンアップを行いましょう。

## 生成ファイルの削除 {#remove-generated-files}

{{< text bash >}}
$ rm -f sample.yaml helloworld-region*.zone*.yaml
{{< /text >}}

## `sample` 名前空間の削除 {#remove-the-sample-namespace}

{{< text bash >}}
$ for CTX in "$CTX_PRIMARY" "$CTX_R1_Z1" "$CTX_R1_Z2" "$CTX_R2_Z3" "$CTX_R3_Z4"; \
  do \
    kubectl --context="$CTX" delete ns sample --ignore-not-found=true; \
 done
{{< /text >}}

**おめでとうございます！** ローカリティ負荷分散タスクを無事に完了しました！
