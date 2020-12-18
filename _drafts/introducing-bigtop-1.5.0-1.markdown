---
layout: post
title:  "Bigtop が提供するパッケージを使って Hadoop クラスタを構築する"
date:   2020-12-19 00:00:00 +0900
category: blog
tags:   apache bigtop oss
---
本記事は,
[Distributed computing (Apache Spark, Hadoop, Kafka, ...) Advent Calendar 2020](https://qiita.com/advent-calendar/2020/distributed-computing)
19日目の記事です。

この記事では, [Apache Bigtop](https://bigtop.apache.org) (以下 Bigtop) が提供する
[deb](https://wiki.debian.org/deb) や [rpm](https://rpm.org/) 形式のパッケージを使って、
Hadoop クラスタを構築する方法を紹介します。
