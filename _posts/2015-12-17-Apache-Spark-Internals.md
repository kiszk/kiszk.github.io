---
layout: post
title: You're up and running!
categories: ['ApacheSpark']
---

[Qiita](http://qiita.com/kiszk/items/53ec7f0419d71790d9af)から引っ越してきました

Apache Sparkの内部構造・動作について説明している情報源を紹介します（今後、随時更新していきたいと思いますので、誤り・他の情報源などありましたらお知らせください）。
（翻訳を除き）全て英語のスライド・文書ですが、長い文が書かれているものは少ないので、理解できるかと思います。


# 全体像
- A Deeper Understanding of Spark’s Internals
  - https://spark-summit.org/2014/wp-content/uploads/2014/07/A-Deeper-Understanding-of-Spark-Internals-Aaron-Davidson.pdf
  - Execution modelとShuffleに絞って解説しています（Cachingについては説明していません）
- Spark Architecture
  - http://0x0fff.com/spark-architecture/
  - 日本語訳はこちら
     - http://qiita.com/kimutansk/items/3496e5b139959e362f02
  - Master&worker, Java heapの使われ方、RDDのpartition、について説明しています
  - 1.6からの新しいMemory管理については、こちら
    - http://0x0fff.com/spark-memory-management/
    - 日本語訳はこちら
      - http://qiita.com/giwa/items/96cc6cc9ea74aae83e2e
- Apache Spark Architecture (Nov 7, 2015)
  - http://www.slideshare.net/AGrishchenko/apache-spark-architecture
  - 114ページと長いですが、Spark全体のコンポーネントを網羅しています。
  - Sparkでプログラムを実行したことがある人が、内部構造を理解しようとするとき、最初に眺めるのに最適なスライドだと思います。
- Introduction to Apache Spark Internal (May 19, 2015)
  - http://www.slideshare.net/michiard/introduction-to-spark-internals
  - DAG scheduler, Task scheduler, Executor, RDD, Block Manager, Shuffleといった構成要素について説明されています
     - Shuffleは、Hash-basedとSort-basedを説明しています
  - RDDの説明が比較的詳しい
- Mastering Apache Spark
  - https://jaceklaskowski.gitbooks.io/mastering-apache-spark/content/
  - Spark全体のコンポーネントを網羅しています。
  - Sparkのプログラムにそって、RDDなどを説明してくれている章（Section 2）もあります。
  - スライドではないので、他に比べて文章が多めです。それでも、クラス間の関係や、実際の実行時の画面キャプチャなど、図絵が多く使われており、わかりやすいと思います。
- Spark Internals (May 29, 2014)
  - http://www.slideshare.net/taroleo/spark-internal-hadoop-source-code-reading-16-in-japan
  - RDDやTaskなど、要所のコンポーネントのソースの一部が紹介されています
  - 全体像を掴まれたあとにこれを読むと、ソースが紹介されているので、こういう実装になっているのか、と理解できるかと思います（ただし、2014年の資料なので、コードが古い部分もあります）
  - ClosureSerializerについて触れている数少ない資料の１つです
- Spark Internals
  - https://github.com/JerryLead/SparkInternals/
  - 日本語訳はこちら
     - http://qiita.com/kimutansk/items/f47c2a28eda6817db531
     - http://qiita.com/kimutansk/items/3496e5b139959e362f02
     - http://qiita.com/kimutansk/items/be5a29c539061ad5a634
     - http://qiita.com/kimutansk/items/e36db0f22893ee4c428d
  - Overview, Job logical plan, Job physical plan, shuffle, Master&worker coordination, cache and check point, broad cast,について詳しく書かれています
  - 実装について、ソースコードやデータ構造の絵を使って説明しているので、細かい点もわかりやすいです
  - 詳細を把握したい人はおすすめです。


#RDD
- Anatomy of RDD
  - スライド　http://www.slideshare.net/datamantra/anatomy-of-rdd
  - ビデオ　http://blog.madhukaraphatak.com/anatomy-of-rdd/
  - データの持ち方、計算のされかた、キャッシュの方法、など説明しています
  - ある程度、ソースコードを読んでからでないと、説明がピンとこないかもしれません。
- caching and checkpoint
  - https://github.com/JerryLead/SparkInternals/blob/master/markdown/english/6-CacheAndCheckpoint.md
  - 全体像、の最後で紹介した資料の6章ですが、cacheについて、概念・実装をデータ構造の絵とともにわかりやすく説明しています
- CacheされないRDDのpartitionに関する説明
  - https://www.quora.com/In-Spark-Where-is-one-RDD-which-hasnt-called-its-cache-or-persist-method-deleted
  - CacheされないRDDのpartitionは、ただのiteratorとそのSeqなので、使い終わったらどこからも参照がなくなってGCで回収される
- Controlling Parallelism in Spark
  - http://www.bigsynapse.com/spark-input-output
  - localファイルから、データを読みだした際に生成される、パーティション数の計算方法の説明です
- Glom in Spark
  - http://blog.madhukaraphatak.com/glom-in-spark/
  - RDDの各パーティションのデータを配列化する、RDD.glom()の説明
- Extending Spark API
  - http://blog.madhukaraphatak.com/extending-spark-api/
  - カスタムRDDの作り方
- Succinct Spark: Queries on Compressed RDDs
  - https://amplab.cs.berkeley.edu/succinct-spark-queries-on-compressed-rdds/	(Succinct Spark)
  - https://github.com/amplab/succinct
  - (RDDの構造の説明とは異なりますが)簡潔データ構造を用いて、圧縮した形でRDD内にデータを持ち、展開することなく演算を行うためのカスタムRDDパッケージの説明とソースコード


# Shuffle
- How Spark Beat Hadoop @ 100 TB Daytona GraySort Challenge 
  - http://www.slideshare.net/cfregly/advanced-apache-spark-meetup-how-spark-beat-hadoop-100-tb-daytona-graysort-challenge
  - P.22-34に、シャッフルの概要、設定できるオプションについて触れています
- Spark Architecture: Shuffle
  - http://0x0fff.com/spark-architecture-shuffle/
  - 日本語訳はこちら
     - http://qiita.com/giwa/items/08ac5bda1eabb8c597b3
     - http://qiita.com/kimutansk/items/3496e5b139959e362f02
  - 絵を用いて動作を説明しています
    - Hash, Sort, Tungsten-sortの３つの動作をそれぞれ説明している資料です 
- Shuffle Process
  - https://github.com/JerryLead/SparkInternals/blob/master/markdown/english/4-shuffleDetails.md
  - 全体像、の最後で紹介した資料の4章ですが、shuffleについて、概念・動作をデータ構造の絵とともにわかりやすく説明しています
    - hashのみを説明しています
- Optimizing Shuffle Performance in Spark
  - http://www.cs.berkeley.edu/~kubitron/courses/cs262a-F13/projects/reports/project16_report.pdf
  - Sparkの(Hash) shuffleの問題点をあげ、改善方法としてcolumn方向のデータ圧縮、送信側がファイルに各内容をまとめてファイル数を減らす、を提案しています。
- Everyday I'm Shuffling - Tips for Writing Better Spark Programs
  - http://www.slideshare.net/databricks/strata-sj-everyday-im-shuffling-tips-for-writing-better-spark-programs
  - Shuffleを使うコードを書くときに、性能を低下させないためのtipsが書かれています
- A simplified sequence diagram of Spark Shuffle Write
  - https://www.linkedin.com/groups/7403611/7403611-6103074141341044737
  - Shuffleの動作のシーケンス図です


# DataFrame
- Anatomy of Data Frame API
  - スライド　http://www.slideshare.net/datamantra/anatomy-of-data-frame-api
  - ビデオ　http://blog.madhukaraphatak.com/anatomy-of-spark-dataframe-api/
  - SQLのData Frame APIに絞って、どのように最適化されるかを説明しています

# Tungsten
- Project Tungsten Nov 12 2015 
  - http://www.slideshare.net/cfregly/advanced-apache-spark-meetup-project-tungsten-nov-12-2015
  - P.37から、Tungstenの３つのアイデアのうち、データ構造の最適化、コード生成、についてより具体的に説明しています
- リンクおきば
  - http://www.slideshare.net/datamantra/anatomy-of-in-memory-processing-in-spark
  - （説明は後日）

# Catalyst
- Data Sources API Cassandra Spark Connector Spark 1.5.1 Zeppelin 0.6.0
  - http://www.slideshare.net/cfregly/advanced-apache-spark-meetup-data-sources-api-cassandra-spark-connector-spark-151-zeppelin-060
  - P.15-18に、簡単にPlan optimizerについて触れています
- SparkSQLInternal
  - http://www.trongkhoanguyen.com/2014/11/introduction-to-sparksql.html
  - SparkSQLがどのようにRDDのオペレーションとなって最適化され実行されるか説明しています。
  - http://www.trongkhoanguyen.com/2015/08/sparksql-internals.html
  - （説明は後日）
  - https://hxquangnhat.com/2015/04/10/sparksql-internals-part-1-sqlcontext/
  - （説明は後日）
  - https://hxquangnhat.com/2015/04/14/arch-sparksql-internals-part-2-sparksql-data-flow/
  - （説明は後日）
  - http://www.slideshare.net/sachinparmarss/deep-dive-spark-data-frames-sql-and-catalyst-optimizer
  - （説明は後日）
- リンクおきば
  - http://www.slideshare.net/maropu0804/20160322-bdi
  - おすすめ
  - http://www.slideshare.net/SparkSummit/enhancements-on-spark-sql-optimizer-by-min-qiu
  - （説明は後日）
  - http://www.slideshare.net/ueshin/introduction-to-spark-sql-catalyst
  - （説明は後日）
  - http://igm.univ-mlv.fr/~ocure/LIGM_LIKE/Teaching/m2Log/sparkSQLDF.pdf
  - （説明は後日）
