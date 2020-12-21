---
layout: post
title:  "Bigtop のスモークテストを使って Hadoop/Spark クラスタの動作確認を行う"
category: blog
tags:   apache bigtop oss
---
本記事は,
[Distributed computing (Apache Spark, Hadoop, Kafka, ...) Advent Calendar 2020](https://qiita.com/advent-calendar/2020/distributed-computing)
21日目の記事です。

[前回の記事](/blog/2020/12/20/introducing-bigtop-1.5.0-2.html) では,
[Apache Bigtop](https://bigtop.apache.org) (以下 Bigtop) が提供する
Puppet マニフェストを使って、Hadoop/Spark クラスタのデプロイを自動化する方法を紹介しました。
本記事では, Bigtop が提供するスモークテストを使うことで、
デプロイしたクラスタに対して簡単な動作確認を行う方法を紹介します。

なお、環境は前回の記事で構築した Ubuntu 18.04 のクラスタを引き続き使用し、
ダウンロード・展開した Bigtop の tarball も、そのまま /mnt/share 以下に
展開されているものとします。


## Bigtop が提供するスモークテスト

スモークテストは
[bigtop-tests/smoke-tests](https://github.com/apache/bigtop/tree/release-1.5.0/bigtop-tests/smoke-tests)
以下に、コンポーネント別に格納されています。

試しに Hive のスモークテストである
[TestHiveSimple.groovy](https://github.com/apache/bigtop/blob/release-1.5.0/bigtop-tests/smoke-tests/hive/TestHiveSimple.groovy),
[passwd.ql](https://github.com/apache/bigtop/blob/release-1.5.0/bigtop-tests/smoke-tests/hive/passwd.ql)
を見ると, /etc/passwd を HDFS にアップロードし、その内容を EXTERNAL TABLE として参照しています。

また, Spark のスモークテストである
[TestSpark.groovy](https://github.com/apache/bigtop/blob/release-1.5.0/bigtop-tests/smoke-tests/spark/TestSpark.groovy)
を見ると, Spark が提供するサンプルコードである
[JavaSparkSQLExample](https://github.com/apache/spark/blob/v2.4.5/examples/src/main/java/org/apache/spark/examples/sql/JavaSparkSQLExample.java)
や [dataframe.R](https://github.com/apache/spark/blob/v2.4.5/examples/src/main/r/dataframe.R) を実行しています。
これらは Java や R から, JSON ファイルをパースして SparkSQL でデータを操作しています。

いずれも、前回行った簡単な確認 (`SELECT 1` という HiveQL や Pi 計算の実行) よりは、
実際のユースケースに近い処理を行っており[^1], 環境構築に問題があった場合に検出できる範囲が広がりそうなので、
これらのテストを実行してみます。

[^1]: テストの手厚さはコンポーネントによってかなりムラがあり, HDFS などではさらに網羅的な機能確認を行っています。


## スモークテストの実行

前回の記事で構築したクラスタをそのまま使い、スモークテストを実行します。
[README](https://github.com/apache/bigtop/blob/release-1.5.0/bigtop-tests/smoke-tests/README)
の説明に従い、最初に `$JAVA_HOME` およびテストするコンポーネントに応じた環境変数を設定します。

```
$ export JAVA_HOME=/usr/lib/jvm/java-8-openjdk-amd64
$ export HIVE_HOME=/usr/lib/hive
$ export HIVE_CONF_DIR=/etc/hive/conf
$ export SPARK_HOME=/usr/lib/spark
```

Bigtop はビルドツールに [Gradle](https://gradle.org/) を使っており、
スモークテストも Gradle タスクとして実行します。
以下のように、複数のコンポーネントに対するテストも一度にまとめて実行できます。

```
$ cd /mnt/share/bigtop-1.5.0
$ ./gradlew bigtop-tests:smoke-tests:hive:test bigtop-tests:smoke-tests:spark:test -Psmoke.tests --info

(略)

> Task :bigtop-tests:smoke-tests:hive:test
Finished generating test XML results (0.015 secs) into: /mnt/share/bigtop-1.5.0/bigtop-tests/smoke-tests/hive/build/test-results/test
Generating HTML test report...
Finished generating test html results (0.036 secs) into: /mnt/share/bigtop-1.5.0/bigtop-tests/smoke-tests/hive/build/reports/tests/test
Now testing...
:bigtop-tests:smoke-tests:hive:test (Thread[Daemon worker,5,main]) completed. Took 24.59 secs.

(略)

> Task :bigtop-tests:smoke-tests:spark:test
Finished generating test XML results (0.003 secs) into: /mnt/share/bigtop-1.5.0/bigtop-tests/smoke-tests/spark/build/test-results/test
Generating HTML test report...
Finished generating test html results (0.025 secs) into: /mnt/share/bigtop-1.5.0/bigtop-tests/smoke-tests/spark/build/reports/tests/test
Now testing...
:bigtop-tests:smoke-tests:spark:test (Thread[Daemon worker,5,main]) completed. Took 1 mins 59.17 secs.

BUILD SUCCESSFUL in 2m 51s
9 actionable tasks: 9 executed
Stopped 1 worker daemon(s).
```

無事、すべてのテストが成功しました。


## スモークテストの結果確認

スモークテストの結果は、上記の出力メッセージ中にあるとおり,
bigtop-tests/smoke-tests/(コンポーネント名)/build/reports/tests/test
に出力されます。
実行したテストの内訳がわかるのに加え、失敗した場合は、こちらからスタックトレース等を確認することもできます。

![hive-test-report.png](/assets/2020-12-21-introducing-bigtop-1.5.0-4/hive-test-report.png)
![spark-test-report.png](/assets/2020-12-21-introducing-bigtop-1.5.0-4/spark-test-report.png)
