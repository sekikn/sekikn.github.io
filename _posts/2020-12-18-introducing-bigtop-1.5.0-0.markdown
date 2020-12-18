---
layout: post
title:  "Apache Bigtop の概要と最新動向"
category: blog
tags:   apache bigtop oss
---
本記事は,
[Distributed computing (Apache Spark, Hadoop, Kafka, ...) Advent Calendar 2020](https://qiita.com/advent-calendar/2020/distributed-computing)
18日目の記事です。

この記事では, [Apache Bigtop](https://bigtop.apache.org) (以下 Bigtop) という OSS プロジェクトの概要と、
2020年12月時点の最新動向について紹介します。

## Bigtop の概要と歴史

Bigtop は, [Apache Hadoop](https://hadoop.apache.org/) エコシステムの環境構築やテストを容易にするための
[Apache Software Foundation](https://apache.org/) 傘下のプロジェクトで、以下のような機能を提供します。

1. Hadoop や [Spark](https://spark.apache.org/) などのビッグデータ関連 OSS を,
   [deb](https://wiki.debian.org/deb) や [rpm](https://rpm.org/) 形式にビルドしたバイナリパッケージ。
2. パッケージのインストールとその後の環境設定 (以下、併せてデプロイと呼びます) を自動化するための
   [Puppet](https://puppet.com/open-source/#osp) マニフェスト。
3. デプロイ後にクラスタの動作を確認するための、スモークテストやインテグレーションテスト。
4. テストや PoC 用途などに、仮想マシンやコンテナを用いた擬似クラスタをローカル環境に構築する provisioner と呼ばれる機能。

Bigtop は、2011年に [Apache incubator](https://incubator.apache.org/) 傘下のプロジェクトとして活動を開始し、
2012 年に TLP (トップレベルプロジェクト) に昇格した歴史あるプロジェクトです。
開始から10年近くに渡り、だいたい半年から1年半くらいの間隔で、継続的に最新版をリリースしてきました。
記事執筆時点での最新バージョンは、2020/12/15 にリリースされた 1.5.0 です。

私は [1.5.0 のリリースマネージャを務めた](https://lists.apache.org/thread.html/r226f6cfe4da0671afda32490adcee42da1c9bcc34eb4da41d604c6cf%40%3Cdev.bigtop.apache.org%3E)
のですが, Bigtop については日本語の情報が少なく、国内での知名度は低いかと思います。
また、英語の [公式サイト](https://bigtop.apache.org) や
[Wiki](https://cwiki.apache.org/confluence/display/BIGTOP/Index) のドキュメントも、
あまり親切でなかったり、古い情報が残っていたりします (これは、おいおい改善していきたいと考えています).
そこで、せっかくリリースした新バージョンを活用してもらえるよう、紹介記事を書こうと思った次第です。

余談ですが, [AWS](https://aws.amazon.com/) のマネージド Hadoop サービスである
[EMR](https://aws.amazon.com/emr/) では、2015年にリリースされた
[バージョン 4.0 から、インストールされるパッケージのビルドに Bigtop を利用しており](https://aws.amazon.com/blogs/aws/elastic-mapreduce-release-4-0-0-with-updated-applications-now-available/),
その仕組みを使うことで,
[ユーザが自分でビルドしたソフトウェアをクラスタ作成時にデプロイすることもできます](https://aws.amazon.com/blogs/big-data/building-and-deploying-custom-applications-with-apache-bigtop-and-amazon-emr/).
また、公式ドキュメント等で言及されているわけではありませんが,
[Microsoft Azure](https://azure.microsoft.com/) の [HDInsight](https://azure.microsoft.com/en-us/services/hdinsight/) や
[Google Cloud](https://cloud.google.com/) の [Dataproc](https://cloud.google.com/dataproc) も,
(少なくとも記事執筆時点では) クラスタの構築に Bigtop の機能を利用しているようです[^1].
ですので、読者の方が意識しないところで、既に Bigtop を利用しているかもしれません。

[^1]: 2020/12 時点の最新バージョンである HDInsight 4.0 および Dataproc 1.5 で確認。
      クラスタに SSH でログインして `apt-cache show hadoop*` を実行すると、
      Maintainer に `Bigtop <dev@bigtop.apache.org>` と表示される。

## Bigtop 1.5.0 の紹介

Bigtop 1.5.0 を使うことで、以下のようなソフトウェア (Bigtop ではコンポーネントと呼びます) を容易に導入することができます。

| コンポーネント | バージョン |
| --------------------- | ---------------- |
| Alluxio               | 1.8.2            |
| Apache Ambari         | 2.6.1            |
| Elasticsearch         | 5.6.14           |
| Apache Flink          | 1.6.4            |
| Apache Flume          | 1.9.0            |
| Apache Giraph         | 1.2.0            |
| Greenplum Database    | 5.10.0           |
| Apache Hadoop         | 2.10.1           |
| Apache HBase          | 1.5.0            |
| Apache Hive           | 2.3.6            |
| Apache Ignite         | 2.7.6            |
| Apache Kafka          | 2.4.0            |
| Kibana                | 5.4.1            |
| Apache Livy           | 0.7.0            |
| Logstash              | 5.4.1            |
| Apache Mahout         | 0.13.0           |
| Apache Oozie          | 4.3.0            |
| Apache Phoenix        | 4.15.0-HBase-1.5 |
| Quantcast File System | 2.0.0            |
| Apache Solr           | 6.6.6            |
| Apache Spark          | 2.4.5            |
| Apache Sqoop          | 1.4.6            |
| Apache Sqoop 2        | 1.99.4           |
| Apache Tez            | 0.9.2            |
| YCSB                  | 0.12.0           |
| Apache Zeppelin       | 0.8.2            |
| Apache Zookeeper      | 3.4.13           |

Hadoop や Spark のバージョンが 2.x 系と、少し古いことに気づかれたかもしれません。
既に [Hadoop を 3.x 系にバージョンアップするためのパッチも投稿されている](https://issues.apache.org/jira/browse/BIGTOP-3280)
ことから、おそらく次のバージョンでは Hadoop, Spark とも 3.x 系を採用するとともに、
それらと互換性のないコンポーネントは削除されることになりそうです。

導入先の環境としては、以下の Linux ディストリビューションおよび CPU アーキテクチャがサポートされます。
CentOS 8, Debian 10, Fedora 31, Ubuntu 18.04 は本バージョンで初めて追加されました。

| Linux ディストリビューション | バージョン |
| ------ | ------------ |
| CentOS | 7, 8         |
| Debian | 9, 10        |
| Fedora | 31           |
| Ubuntu | 16.04, 18.04 |

| CPU アーキテクチャ |
| --------------- |
| x86_64/amd64    |
| aarch64/arm64   |

次回以降の記事では、上で紹介した 1~4 の機能について、実際の使用方法を説明したいと思います。
