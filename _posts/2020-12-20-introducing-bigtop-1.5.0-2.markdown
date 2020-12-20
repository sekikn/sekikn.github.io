---
layout: post
title:  "Bigtop の Puppet マニフェストを使って Hadoop/Spark クラスタの構築を自動化する"
date: 2020-12-20
category: blog
tags:   apache bigtop oss
---
本記事は,
[Distributed computing (Apache Spark, Hadoop, Kafka, ...) Advent Calendar 2020](https://qiita.com/advent-calendar/2020/distributed-computing)
20日目の記事です。

[前回の記事](/blog/2020/12/19/introducing-bigtop-1.5.0-1.html) では,
[Apache Bigtop](https://bigtop.apache.org) (以下 Bigtop) が提供するビルド済みのパッケージを使うことで,
apt や yum, dnf といったパッケージマネージャ経由で Hadoop をインストールしました。
しかし、導入すべきコンポーネントが多数ある場合、一つ一つインストールと環境設定を行うのは大変です。
そこで今回は, Bigtop が提供する Puppet[^1] マニフェスト[^2] を使うことで、デプロイを自動化する方法を紹介します。

[^1]: Puppet は、[Puppet 社](https://puppet.com/) が開発している構成管理ツールです。
      今回はその OSS 版である [Open Source Puppet](https://puppet.com/open-source/#osp) を使います。

[^2]: サーバのあるべき状態を記述した設定ファイル。

なお、想定する環境は
[前回](/blog/2020/12/19/introducing-bigtop-1.5.0-1.html#想定する環境)
と同様ですが、説明を簡単にするため、全ノードがネットワーク上の共有ストレージを
/mnt/share にマウントしているものとします。
また、せっかくなので色々な Linux ディストリビューションでの実行例を示すため、
今回は Ubuntu 18.04 を使います。


## Bigtop の Puppet マニフェストのダウンロードと, Puppet のインストール

最初に, Bigtop が提供する Puppet マニフェストを含む tarball をダウンロードします。
[Bigtop の公式サイト](https://bigtop.apache.org/) から "Latest release" をクリックし、
遷移先のページで "The latest release of Apache Bigtop software framework"
と書かれたファイルをダウンロードして、共有ディレクトリに展開します。

![link-to-the-latest-release](/assets/2020-12-20-introducing-bigtop-1.5.0-2/link-to-the-latest-release.png)

※任意のノードで実行
```
$ cd /mnt/share
$ curl -O https://downloads.apache.org/bigtop/bigtop-1.5.0/bigtop-1.5.0-project.tar.gz
$ tar xf bigtop-1.5.0-project.tar.gz
$ cd bigtop-1.5.0
```

次に、全ノードに Puppet をインストールします。
上でダウンロードしたファイルの中にインストール用のスクリプトがあるので、
各ノードで以下のコマンドを実行します。

※全てのノードで実行
```
$ sudo bigtop_toolchain/bin/puppetize.sh
```

本コマンドは, Debian 系のディストリビューション (Debian, Ubuntu) では標準リポジトリから,
Red Hat 系 (CentOS, Fedora) では EPEL から、Puppet と必要な追加モジュールをインストールします。


## Hiera の設定と, Puppet マニフェストの適用

それでは, Puppet マニフェストを適用し、パッケージのインストールと設定を行いましょう。
Puppet マニフェストは展開先ディレクトリの bigtop-deploy/puppet 以下にあります。

ここで、インストールするパッケージや各種の設定値を指定するため、
bigtop-deploy/puppet/hieradata/site.yaml[^3] を以下のように編集します[^4].

[^3]: この YAML は, Puppet で変数を管理する仕組みである
      [Hiera](https://puppet.com/docs/puppet/5.5/hiera.html) の設定ファイルです。

[^4]: 今回は最小限の設定のみ記述していますが、他にどういった変数が設定できるかは、
      bigtop-deploy/puppet/hieradata/bigtop 以下の YAML ファイルに
      デフォルト値が定義されていますので、必要に応じてご参照ください。


```
$ cat bigtop-deploy/puppet/hieradata/site.yaml
---
bigtop::hadoop_head_node: "(マスターノードの FQDN)"

hadoop::hadoop_storage_dirs:
  - /data

hadoop_cluster_node::cluster_components:
  - hdfs
  - yarn
  - mapreduce
  - hive
  - spark
```

これらの変数の意味は以下のとおりです。

* `bigtop::hadoop_head_node`: マスターノードの FQDN を指定します。
  自身の FQDN がこの値に一致したノードでは, NameNode や ResourceManager など、マスターノードで動作するサービスが、
  そうでないノードでは, DataNode や NodeManager など、ワーカーノードで動作するサービスがインストールされます。

* `hadoop::hadoop_storage_dirs`: HDFS の格納領域に使用する、ローカルディレクトリのパスを指定します。

* `hadoop_cluster_node::cluster_components`: デプロイしたいコンポーネントを配列として列挙します。
  今回は Hadoop に加えて, Hive と Spark もインストールしてみます。

必要な情報を site.yaml に定義したら、Hiera 関連のファイル一式を
ローカルにコピーし, Puppet マニフェストを適用します。

※全ノードで実行
```
$ sudo cp -r /mnt/share/bigtop-1.5.0/bigtop-deploy/puppet/hiera* /etc/puppet/
$ sudo puppet apply --modulepath=bigtop-deploy/puppet/modules:/usr/share/puppet/modules bigtop-deploy/puppet/manifests

(略)

Notice: Applied catalog in 1049.77 seconds
$ echo $?
0
```

`puppet apply` コマンドを実行すると、いくつかの Warning が出力されるかもしれませんが、
上記のようにステータスコードが 0 で終了していれば問題ありません[^5].

[^5]: Bigtop の Puppet マニフェストは、Puppet 3.x から 6.x と、幅広いバージョンに対応しているため、
      将来的に廃止される機能を使っているとみなされ、警告が表示される場合があります。

なお、2つ目のコマンドについては、少し注意点があります。

* Bigtop の Puppet マニフェストは、一部 Puppet 4 から導入された記法を使用しているため、
  Puppet 3.x 系がインストールされる CentOS 7 および Ubuntu 16.04 ではこのまま実行できません。
  これらの Linux ディストリビューションでは, `--parser=future` オプションを追加で指定してください。

* `--modulepath` オプションに指定したパスの2つ目 (上記コマンドでは `/usr/share/puppet/modules`)
  には, Puppet の追加モジュールのインストール先ディレクトリを指定します
  (このパスは `sudo puppet module list` コマンドで確認できます).
  Debian 系ディストリビューションの場合は上記の値でよいのですが、Red Hat 系の場合は
  CentOS 7 が `/etc/puppet/modules`, CentOS 8 が `/etc/puppetlabs/code/modules` など、
  バージョンによって変わるので、適切な値に書き換えてください。

無事マニフェストの適用が成功すれば、Hadoop クラスタに加えて Hive, Spark もデプロイされているはずです。
念のため、これらのジョブを実行してみます。

※マスターノードで実行
```
$ beeline -u jdbc:hive2://localhost:10000/ -n hive -e 'SELECT 1;'

(略)

Connecting to jdbc:hive2://localhost:10000/
Connected to: Apache Hive (version 2.3.6)
Driver: Hive JDBC (version 2.3.6)
Transaction isolation: TRANSACTION_REPEATABLE_READ
+------+
| _c0  |
+------+
| 1    |
+------+
1 row selected (1.76 seconds)
Beeline version 2.3.6 by Apache Hive
Closing: 0: jdbc:hive2://localhost:10000/
```
```
$ /usr/lib/spark/bin/run-example --master yarn SparkPi 1 1

(略)

20/12/20 15:03:56 INFO SparkContext: Running Spark version 2.4.5
20/12/20 15:03:56 INFO SparkContext: Submitted application: Spark Pi

(略)

20/12/20 15:04:16 INFO Client: Application report for application_1608474329273_0001 (state: ACCEPTED)
20/12/20 15:04:17 INFO Client: Application report for application_1608474329273_0001 (state: RUNNING)

(略)

Pi is roughly 3.1331513315133153
```

問題なく実行できました。


## 最後に

この記事では、Bigtop が提供する Puppet マニフェストを活用して、データ基盤をさらに容易に構築する方法をご紹介しました。

今回は説明をわかりやすくするため、全ノードでコマンドを実行しましたが、実行するコマンドは共通なので,
Parallel SSH などを使って全ノードで同一のコマンドを実行することで、さらに構築作業を自動化できます。
以下にコマンド実行例を示します。

※全ノードにパスワード無しで SSH ログイン可能なノードから, tarball のダウンロード・展開と site.yaml の編集を行ってから実行
```
$ sudo apt-get install -y pssh
$ parallel-ssh -H '(全ノード名をスペース区切りで列挙)' 'sudo /mnt/share/bigtop-1.5.0/bigtop_toolchain/bin/puppetize.sh'
$ parallel-ssh -H '(全ノード名をスペース区切りで列挙)' 'sudo cp -r /mnt/share/bigtop-1.5.0/bigtop-deploy/puppet/hiera* /etc/puppet/'
$ parallel-ssh -t0 -H '(全ノード名をスペース区切りで列挙)' 'sudo puppet apply --modulepath=/mnt/share/bigtop-1.5.0/bigtop-deploy/puppet/modules:/usr/share/puppet/modules /mnt/share/bigtop-1.5.0/bigtop-deploy/puppet/manifests'
```
