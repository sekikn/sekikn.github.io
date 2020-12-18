---
layout: post
title:  "Bigtop の provisioner を使って仮想マシンやコンテナ上に Hadoop/Spark クラスタを構築する"
date:   2020-12-21 00:00:00 +0900
category: blog
tags:   apache bigtop oss
---
本記事は,
[Distributed computing (Apache Spark, Hadoop, Kafka, ...) Advent Calendar 2020](https://qiita.com/advent-calendar/2020/distributed-computing)
21日目の記事です。

この記事では, [Apache Bigtop](https://bigtop.apache.org) (以下 Bigtop) が提供する
provisioner という機能を使って、仮想マシンやコンテナ上に Hadoop/Spark クラスタを構築する方法を紹介します。
