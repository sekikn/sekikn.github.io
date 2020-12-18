---
layout: post
title:  "Bigtop の Puppet マニフェストを使って Hadoop/Spark クラスタの構築を自動化する"
category: blog
tags:   apache bigtop oss
---
本記事は,
[Distributed computing (Apache Spark, Hadoop, Kafka, ...) Advent Calendar 2020](https://qiita.com/advent-calendar/2020/distributed-computing)
20日目の記事です。

この記事では, [Apache Bigtop](https://bigtop.apache.org) (以下 Bigtop) が提供する
[Puppet](https://puppet.com/open-source/#osp) マニフェストを使って、
Hadoop/Spark クラスタの構築を自動化する方法を紹介します。
