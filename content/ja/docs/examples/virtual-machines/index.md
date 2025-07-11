---
title: 仮想マシン上で Bookinfo アプリケーションをデプロイする
description: メッシュ内の仮想マシンで動作する MySQL サービスを使って Bookinfo アプリケーションを実行します。
weight: 60
keywords:
  - virtual-machine
  - vms
aliases:
  - /ja/docs/examples/integrating-vms/
  - /ja/docs/examples/mesh-expansion/bookinfo-expanded
  - /ja/docs/examples/virtual-machines/bookinfo/
  - /ja/docs/examples/vm-bookinfo
owner: istio/wg-environments-maintainers
test: yes
---

この例では、仮想マシン（VM）上でサービスを動作させることで、Kubernetes をまたいで Bookinfo アプリケーションをデプロイし、単一のメッシュとしてこのインフラを制御する方法を示します。

## 概要 {#overview}

{{< image width="80%" link="./vm-bookinfo.svg" caption="仮想マシン上で Bookinfo を実行" >}}

<!-- source of the drawing
https://docs.google.com/drawings/d/1G1592HlOVgtbsIqxJnmMzvy6ejIdhajCosxF1LbvspI/edit
-->

## 始める前に {#before-you-begin}

- [仮想マシンインストールガイド](/ja/docs/setup/install/virtual-machine/)に従って Istio をセットアップしてください。

- [Bookinfo](/ja/docs/examples/bookinfo/) サンプルアプリケーションを（`bookinfo` 名前空間に）デプロイしてください。

- [仮想マシンの設定](/ja/docs/setup/install/virtual-machine/#configure-the-virtual-machine)に従い、仮想マシンを作成し `vm` 名前空間に追加してください。

## 仮想マシン上で MySQL を動かす {#running-MySQL-on-the-VM}

仮想マシン上に MySQL をインストールし、ratings サービスのバックエンドとして設定します。

以下のコマンドはすべて仮想マシン上で実行します。

`mariadb` をインストール：

{{< text bash >}}
$ sudo apt-get update && sudo apt-get install -y mariadb-server
$ sudo sed -i '/bind-address/c\bind-address = 0.0.0.0' /etc/mysql/mariadb.conf.d/50-server.cnf
{{< /text >}}

認証情報を設定：

{{< text bash >}}
$ cat <<EOF | sudo mysql

# root へのアクセス権を付与

GRANT ALL PRIVILEGES ON _._ TO 'root'@'localhost' IDENTIFIED BY 'password' WITH GRANT OPTION;

# 他の IP からの root アクセス権を付与

CREATE USER 'root'@'%' IDENTIFIED BY 'password';
GRANT ALL PRIVILEGES ON _._ TO 'root'@'%' WITH GRANT OPTION;
FLUSH PRIVILEGES;
quit;
EOF
$ sudo systemctl restart mysql
{{< /text >}}

MySQL の詳細な設定方法は [Mysql](https://mariadb.com/kb/en/library/download/) を参照してください。

仮想マシン上で ratings データベースを mysql にインポートします。

{{< text bash >}}
$ curl -LO {{< github_file >}}/samples/bookinfo/src/mysql/mysqldb-init.sql
$ mysql -u root -ppassword < mysqldb-init.sql
{{< /text >}}

Bookinfo アプリケーションの出力の違いを直感的に確認するため、以下のコマンドで
`ratings` データベースを変更・確認できます：

{{< text bash >}}
$ mysql -u root -ppassword test -e "select \* from ratings;"
+----------+--------+
| ReviewID | Rating |
+----------+--------+
| 1 | 5 |
| 2 | 4 |
+----------+--------+
{{< /text >}}

`ratings` データベースを変更：

{{< text bash >}}
$ mysql -u root -ppassword test -e "update ratings set rating=1 where reviewid=1;select \* from ratings;"
+----------+--------+
| ReviewID | Rating |
+----------+--------+
| 1 | 1 |
| 2 | 4 |
+----------+--------+
{{< /text >}}

## mysql サービスをメッシュに公開する {#expose-the-mysql-service-to-the-mesh}

仮想マシンが起動すると、自動的にメッシュに登録されます。
ただし、Pod を作成する場合と同様に、簡単にアクセスできるよう Service を作成する必要があります。

{{< text bash >}}
$ cat <<EOF | kubectl apply -f - -n vm
apiVersion: v1
kind: Service
metadata:
name: mysqldb
labels:
app: mysqldb
spec:
ports:

- port: 3306
  name: tcp
  selector:
  app: mysqldb
  EOF
  {{< /text >}}

## mysql サービスの利用 {#using-the-mysql-service}

Bookinfo の ratings サービスはこの仮想マシン上のデータベースを利用します。
正常に動作するか確認するため、仮想マシン上の mysql データベースを使う ratings サービスの第 2 バージョンを作成します。
次に、review サービスが ratings サービスの第 2 バージョンを使うようルーティングルールを指定します。

{{< text bash >}}
$ kubectl apply -n bookinfo -f @samples/bookinfo/platform/kube/bookinfo-ratings-v2-mysql-vm.yaml@
{{< /text >}}

Bookinfo が ratings バックエンドを使うようにするルーティングルールを作成：

{{< text bash >}}
$ kubectl apply -n bookinfo -f @samples/bookinfo/networking/virtual-service-ratings-mysql-vm.yaml@
{{< /text >}}

Bookinfo アプリケーションの出力で Reviewer1 が 1 つ星、Reviewer2 が 4 つ星になっているか、
または仮想マシン上の ratings サービスを変更して結果を確認できます。

## 仮想マシンから Kubernetes サービスへアクセス {#reaching-Kubernetes-services-from-the-virtual-machine}

上記の例では、仮想マシンを 1 つのサービスとして扱いました。
仮想マシンから Kubernetes のサービスをシームレスに呼び出すこともできます：

{{< text bash >}}
$ curl productpage.bookinfo:9080
...

<title>Simple Bookstore App</title>
...
{{< /text >}}

Istio の [DNS プロキシ](/ja/docs/ops/configuration/traffic-management/dns-proxy/)は、仮想マシン用に DNS を自動設定し、Kubernetes のホスト名でアクセスできるようにします。

## クリーンアップ {#cleanup}

- [`Bookinfo` のクリーンアップ](/ja/docs/examples/bookinfo/#cleanup)の手順に従い、`Bookinfo` サンプルアプリとその設定を削除します。
- `mysqldb` Service を削除：

  {{< text syntax=bash snip_id=none >}}
  $ kubectl delete service mysqldb
  {{< /text >}}

- [仮想マシンのアンインストール](/ja/docs/setup/install/virtual-machine/#uninstall)の手順に従い、VM をクリーンアップします。
