---
layout: post
title:  "Bigtop の provisioner を使って仮想マシンやコンテナ上に Hadoop/Spark クラスタを構築する"
date: 2020-12-22
category: blog
tags:   apache bigtop oss
---
本記事は,
[Distributed computing (Apache Spark, Hadoop, Kafka, ...) Advent Calendar 2020](https://qiita.com/advent-calendar/2020/distributed-computing)
22日目の記事です。

前回までの記事では,
[Apache Bigtop (以下 Bigtop) が提供するパッケージ](/blog/2020/12/19/introducing-bigtop-1.5.0-1.html)と
[Puppet マニフェスト](/blog/2020/12/20/introducing-bigtop-1.5.0-2.html)
を使って Hadoop/Spark クラスタを構築し、そのクラスタを
[スモークテストで動作確認する](/blog/2020/12/21/introducing-bigtop-1.5.0-3.html)
方法を紹介しました。

ただ、いきなりインストール先のサーバを用意する前に、
まずは手元で仮想的なクラスタを構築し、クラスタそのものや
アプリケーションの動作をあらかじめ確認したい、というニーズは多いかと思います。

そこで本記事では, Bigtop が提供する
[provisioner](https://github.com/apache/bigtop/tree/release-1.5.0/provisioner)
という機能を使って、仮想マシン (以下 VM) やコンテナ上に
Hadoop/Spark クラスタを構築する方法を紹介します。
Bigtop は, Vagrant provisioner と Docker provisioner という、
2種類の provisioner を提供しますが、まずは前者から紹介します。

なお, provisioner の実行には、
[前々回の記事](/blog/2020/12/20/introducing-bigtop-1.5.0-2.html)
でダウンロードした Bigtop 1.5.0 を使いますので、
まだダウンロードしていない場合は、以下の手順で適当なディレクトリに展開してください。

```
$ curl -O https://downloads.apache.org/bigtop/bigtop-1.5.0/bigtop-1.5.0-project.tar.gz
$ tar xf bigtop-1.5.0-project.tar.gz 
$ cd bigtop-1.5.0
```


## Vagrant provisioner の使用方法

Vagrant provisioner は、その名のとおり [Vagrant](https://www.vagrantup.com/)
(と、その裏で実際に仮想化を行う [VirtualBox](https://www.virtualbox.org/))
によって作成された VM 上に, Hadoop/Spark クラスタを構築する機能です。

まずはそれぞれの公式サイトの説明に従い, VirtualBox と Vagrant をインストールします。
以下の説明は, VirtualBox 6.1.10 および Vagrant 2.2.6 で動作を確認しました。

その後,
[provisioner/vagrant/README.md の説明](https://github.com/apache/bigtop/tree/release-1.5.0/provisioner/vagrant/README.md#usage)
に従い, vagrant-hostmanager と vagrant-cachier の2つのプラグインをインストールします。

```
$ vagrant plugin install vagrant-hostmanager
$ vagrant plugin install vagrant-cachier
```

Vagrant provisioner を実行するため,
provisioner/vagrant に移動し, YAML 形式の設定ファイルを確認します。

```
$ cd provisioner/vagrant
$ cat vagrantconfig.yaml

(略)

memory_size: 4096
number_cpus: 1
box: "puppetlabs/centos-7.2-64-nocm"
repo: "http://repos.bigtop.apache.org/releases/1.3.0/centos/7/x86_64"
num_instances: 1
distro: centos
components: [hdfs, yarn, mapreduce]
enable_local_repo: false
run_smoke_tests: false
smoke_test_components: [hdfs, yarn, mapreduce]
```

設定ファイル中の repo に, Bigtop の公開リポジトリの URL[^1] を指定します。
以下のように、最新の 1.5.0 リポジトリを使用するよう URL を修正して, VM を起動します。

[^1]: リポジトリの URL の確認方法は
      [12/19 の記事](/blog/2020/12/19/introducing-bigtop-1.5.0-1.html)
      を参照してください。ダウンロードしたリポジトリ定義ファイル
      (.repo もしくは .list) 中に URL が記載されています。

```
$ cat vagrantconfig.yaml | grep ^repo
repo: "http://repos.bigtop.apache.org/releases/1.5.0/centos/7/x86_64"
$ vagrant up

(略)

==> bigtop1: Notice: Finished catalog run in 326.64 seconds
==> bigtop1: Configuring cache buckets...
```

出力されるメッセージを眺めていると、
[前々回の記事](/blog/2020/12/20/introducing-bigtop-1.5.0-2.html)
で紹介した Puppet マニフェストを使って, YAML 中の components に指定した
パッケージ (今回は HDFS, YARN, MapReduce) がデプロイされているのが観察できます。

VM にログインして、Java プロセスを確認すると、NameNode, DataNode,
ResourceManager, NodeManager 等が動作していることが確認できます。

```
$ vagrant ssh
[vagrant@bigtop1 ~]$ sudo jps
14000 JobHistoryServer
14400 NameNode
13719 ResourceManager
14167 NodeManager
13626 WebAppProxyServer
16171 Jps
14636 DataNode
```

試しに pi 計算を実行してみます。

```
[vagrant@bigtop1 ~]$ sudo -u yarn yarn jar /usr/lib/hadoop-mapreduce/hadoop-mapreduce-examples.jar pi 1 1
Number of Maps  = 1
Samples per Map = 1
Wrote input for Map #0
Starting Job

(略)

Job Finished in 25.811 seconds
Estimated value of Pi is 4.00000000000000000000
```

問題なく実行できました。

VM の停止・破棄も、通常の Vagrant の操作と同様、
それぞれ `vagrant halt`, `vagrant destroy` で行います。
また、構築するクラスタの台数を増やしたい場合は
設定ファイル中の num_instances の値を、
デプロイするコンポーネントを変更する場合は components を、
それぞれ変更してください。

さて、ここまで Vagrant provisioner を紹介してきましたが、
現在はあまり活発にメンテナンスされていません。
実際、現状の Vagrant provisioner の作りでは、
上で試した CentOS 7 以外の OS では動作しないはずなので、
それ以外の OS でも動作するよう修正したいと考えています
([BIGTOP-3469](https://issues.apache.org/jira/browse/BIGTOP-3469)).

代わりに、現在よく使われているのが、次に紹介する Docker provisioner です。


## Docker provisioner

Docker provisioner は, VM ではなく Docker コンテナ上に Hadoop/Spark クラスタを構築します。
動作には [Docker](https://www.docker.com/) と
[docker-compose](https://docs.docker.com/compose/) が必要なので、
それぞれの公式サイトの説明に従い、あらかじめインストールしておいてください。

本機能のディレクトリである provisioner/docker に移動し、
設定ファイルの一覧を見ると, OS ごとにデフォルトのものが用意されていることがわかります。

```
$ cd provisioner/docker
$ ls -l
total 56
-rw-r--r-- 1 sekikn sekikn  1060 Dec  7 00:21 config_centos-7.yaml
-rw-r--r-- 1 sekikn sekikn  1060 Dec  7 00:21 config_centos-8.yaml
-rw-r--r-- 1 sekikn sekikn  1060 Dec  7 00:21 config_debian-10.yaml
-rw-r--r-- 1 sekikn sekikn  1058 Dec  7 00:21 config_debian-9.yaml
-rw-r--r-- 1 sekikn sekikn  1062 Dec  7 00:21 config_fedora-31.yaml
-rw-r--r-- 1 sekikn sekikn  1067 Dec  7 00:21 config_ubuntu-16.04.yaml
-rw-r--r-- 1 sekikn sekikn  1067 Dec  7 00:21 config_ubuntu-18.04.yaml
lrwxrwxrwx 1 sekikn sekikn    20 Dec  7 00:21 config.yaml -> config_centos-7.yaml
-rw-r--r-- 1 sekikn sekikn  1130 Dec  7 00:21 docker-compose.yml
-rwxr-xr-x 1 sekikn sekikn 14698 Dec  7 00:21 docker-hadoop.sh
-rw-r--r-- 1 sekikn sekikn  4217 Dec  7 00:21 README.md
```

今回は以下の設定ファイルを使って, Fedora 31 のコンテナ上に Hadoop をデプロイしてみます。

```
$ cat config_fedora-31.yaml 

(略)

docker:
        memory_limit: "4g"
        image: "bigtop/puppet:1.5.0-fedora-31"

repo: "http://repos.bigtop.apache.org/releases/1.5.0/fedora/31/$basearch"
distro: centos
components: [hdfs, yarn, mapreduce]
enable_local_repo: false
smoke_test_components: [hdfs, yarn, mapreduce]
```

最小限の実行コマンドは以下のとおりです.
`-C` で使用する設定ファイルを, `-c` でクラスタの台数を指定します。

```
$ ./docker-hadoop.sh -C config_fedora-31.yaml -c 1

(略)

Notice: Applied catalog in 366.58 seconds
```

正常終了すると、名前に "bigtop" を含むコンテナが作られていることが確認できます。

```
$ docker ps
CONTAINER ID        IMAGE                           COMMAND             CREATED             STATUS              PORTS               NAMES
61bc766a93c5        bigtop/puppet:1.5.0-fedora-31   "/sbin/init"        12 minutes ago      Up 12 minutes                           20201221_111813_r18389_bigtop_1
```

コンテナ上で bash を実行し, Java プロセスを確認したりサンプルジョブを実行したりしてみます。
問題なく動作しているようです。

```
$ docker exec -it 20201221_111813_r18389_bigtop_1 bash
[root@61bc766a93c5 /]# cat /etc/redhat-release 
Fedora release 31 (Thirty One)
[root@61bc766a93c5 /]# jps
1761 NodeManager
566 ResourceManager
1976 Jps
906 JobHistoryServer
1451 DataNode
1035 WebAppProxyServer
1247 NameNode
[root@61bc766a93c5 /]# sudo -u yarn yarn jar /usr/lib/hadoop-mapreduce/hadoop-mapreduce-examples.jar pi 1 1
Number of Maps  = 1
Samples per Map = 1
Wrote input for Map #0
Starting Job

(略)

Job Finished in 18.447 seconds
Estimated value of Pi is 4.00000000000000000000
```

また, Docker provisioner では、YAML ファイルを書き換える以外に、
コマンドラインから引数でパラメータを指定することもでき、使いやすさが向上しています。
試しに、Debian 10 をベースに、クラスタ台数を4台に増やし、デプロイするコンポーネントも
Hadoop に加えて ZooKeeper と HBase を追加してみましょう。
`-d` で既存のコンテナを破棄し, `-k` にインストールするコンポーネントを列挙します。

```
$ ./docker-hadoop.sh -d -C config_debian-10.yaml -c 4 -k 'zookeeper, hdfs, yarn, hbase'
Stopping 20201221_111813_r18389_bigtop_1 ... done
Going to remove 20201221_111813_r18389_bigtop_1
Removing 20201221_111813_r18389_bigtop_1 ... done

(略)

Notice: Applied catalog in 532.54 seconds
```

コンテナの一覧を表示すると、確かに4ノード分起動しています。

```
sekikn@sekikn-ThinkCentre-M75q-Gen-2:~/bigtop-1.5.0/provisioner/docker$ docker ps
CONTAINER ID        IMAGE                           COMMAND             CREATED             STATUS              PORTS               NAMES
df714eb4ad53        bigtop/puppet:1.5.0-debian-10   "/sbin/init"        10 minutes ago      Up 10 minutes                           20201221_113938_r11898_bigtop_2
0e66e4dcb919        bigtop/puppet:1.5.0-debian-10   "/sbin/init"        10 minutes ago      Up 10 minutes                           20201221_113938_r11898_bigtop_4
1091aee58456        bigtop/puppet:1.5.0-debian-10   "/sbin/init"        10 minutes ago      Up 10 minutes                           20201221_113938_r11898_bigtop_3
b3c45a27086c        bigtop/puppet:1.5.0-debian-10   "/sbin/init"        10 minutes ago      Up 10 minutes                           20201221_113938_r11898_bigtop_1
```

コンテナにアクセスして HBase の基本的な操作を実行してみると、確かに HBase がデプロイされていることが確認できます。

```
$ docker exec -it 20201221_113938_r11898_bigtop_1 bash
root@b3c45a27086c:/#  hbase shell
OpenJDK 64-Bit Server VM warning: Using incremental CMS is deprecated and will likely be removed in a future release
HBase Shell
Use "help" to get list of supported commands.
Use "exit" to quit this interactive shell.
Version 1.5.0, rUnknown, Sat Nov 28 13:08:01 UTC 2020

hbase(main):001:0> create 'test', 'cf'
0 row(s) in 1.5680 seconds

=> Hbase::Table - test
hbase(main):002:0> list 'test'
TABLE                                                                                                                                                                                                                     
test                                                                                                                                                                                                                      
1 row(s) in 0.0230 seconds

=> ["test"]
hbase(main):003:0> put 'test', 'row1', 'cf:a', 'value1'
0 row(s) in 0.0930 seconds

hbase(main):004:0> put 'test', 'row2', 'cf:b', 'value2'
0 row(s) in 0.0120 seconds

hbase(main):005:0> put 'test', 'row3', 'cf:c', 'value3'
0 row(s) in 0.0090 seconds

hbase(main):006:0> scan 'test'
ROW                                                     COLUMN+CELL                                                                                                                                                       
 row1                                                   column=cf:a, timestamp=1608521791563, value=value1                                                                                                                
 row2                                                   column=cf:b, timestamp=1608521798779, value=value2                                                                                                                
 row3                                                   column=cf:c, timestamp=1608521806451, value=value3                                                                                                                
3 row(s) in 0.0290 seconds
```
```
root@b3c45a27086c:/# hbase hbtop
OpenJDK 64-Bit Server VM warning: Using incremental CMS is deprecated and will likely be removed in a future release

HBase hbtop - 03:40:14
Version: 1.5.0
Cluster ID: 231c00c8-5fa8-4240-b551-bed52fc07f81
RegionServer(s): 3 total, 3 live, 0 dead
RegionCount: 3 total, 0 rit
Average Cluster Load: 1.00
Aggregate Request/s: 0

NAMESPACE TABLE     REGION                           RS                   #REQ/S  #READ/S #WRITE/S         SF  #SF MEMSTORE LOCALITY 
hbase     namespace 0a77767f0679c66f1fbb220e6fcb3efd b3c45a27086c:16020        0        0        0       0.0B    1     0.0B      1.0 
default   test      d1da1249633f6bf8799e02f6b101477a df714eb4ad53:16020        0        0        0       0.0B    1     0.0B      1.0 
hbase     meta      1588230740                       1091aee58456:16020        0        0        0       0.0B    1     0.0B      1.0 
```


## 最後に

以上のように, Bigtop が提供する provisioner を利用すると、VM やコンテナを利用して、
ローカル環境に簡易的なクラスタを容易に構築することができます。
ぜひ、クラスタやアプリケーションの動作確認、テストや PoC などにご活用ください。
