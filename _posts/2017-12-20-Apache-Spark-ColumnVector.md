---
layout: post
title: Apache Sparkのcolumnar storageの解説
categories: ['ApacheSpark']
tags: ['spark']
published: true
---

# この記事は？

この記事は、[Data Platform Advent Calendar 2017](https://qiita.com/advent-calendar/2017/dataplatform)の20日目の記事です。

# この記事の内容は？

2018年の早い時期にリリース予定のApache Spark 2.3に入る、Columnar storage（Spark内部では`ColumnVector`クラスで定義されています）の改善に関するまとめです。

# 結局何がいいたいの？

- `ColumnVector`は、Spark 2.2までは内部構造体として使われていました。Apache Parquetのデータを読み込んだ際の内部的なデータ置き場、に使われていました。  
Spark 2.3では、下記のような拡張があります。
- `ColumnVector`は、publicなデータ構造とAPIになります。
- `ColumnVector`は、Apache Arrowとのデータのやり取りにも使われます。
- Pandasのデータとのやり取りが高速になります。つまり、Pandasを使ったpysparkの実行が高速になります。
- `DataFrame/Dataset.cache()`のインメモリキャッシュのデータ読み出しも`ColumnVector`を使うことによって効率的になり、`cache()`を使ったプログラムの実行が高速化されます。


# 本論

## 列単位のデータ保存について

Sparkでは、アプリケーションプログラマがプログラムを書く・データを保存する際処理の単位として、行(row）が基本となっています。一方、データベースの分野では、長年の研究・実装の経験から、列（column）単位でデータを保存する、処理するのが有効である場合が多い、ことが知られています。例えば、下記のようなメリットがあります。
1. 列方向にデータを見ると、同じデータ（0など）が存在することが多いので圧縮しやすい
1. 列方向に同じ型のデータが並んでいるので、同じ種類の処理について、並列処理可能なチャンスが存在する

Sparkでは、2.2までに内部的には、下記のような場合に列単位のデータ構造を使ってデータ保存を行っていました。
1. Apache Parquetのデータを読み込んだ際の、内部的なデータ置き場
1. DataFrame/Dataset.cache()を実行した場合の、インメモリキャッシュのデータ格納先（この場合、圧縮も行われます）

Spark 2.3では、列データを使うことによる利点を、いろいろな形でさらに活用しようとしています。


## Apache Spark 2.3では何が変わるの？

### ColumnVectorクラスの公開

Spark 2.3では、Spark summit west 2017で[意向を表明したように](https://www.slideshare.net/databricks/building-robust-etl-pipelines-with-apache-spark/39)ように、列データフォーマットのための[`ColumnVector`](https://github.com/apache/spark/blob/master/sql/core/src/main/java/org/apache/spark/sql/execution/vectorized/ColumnVector.java)クラスを、public APIとして公開する予定です。このクラスは今までも存在していましたが、内部クラスとして使われていて、公開されていませんでした。[topic JIRAエントリ](https://issues.apache.org/jira/browse/SPARK-20960)も作られ、着々と作業が進んでいます。
これによって、オレオレデータ構造を、abstractな`ColumnVector`クラスの実装クラスとして書くことができれば、Sparkのプログラムから低オーバヘッドでデータの読み出しができます。なお、`ColumnVector`クラスはデータ読み出しだけをサポートしていて、書込みはサポートしていません。

### Apache ArrowのサポートとPySparkの高速化

これまで、ColumnVectorを使った外部読み込みのデータは、Apache Parquetだけをサポートしていました（Apache ORCもcolumn storageですが、現在のSpark(2.3でも)の内部では行データに変換して扱っています）。Spark 2.3では、[Apache Arrow]([https://arrow.apache.org/)というインメモリの列データフォーマットもサポートします（Apache Arrowに関しては、[こちら](https://www.slideshare.net/MapR_Japan/apache-arrow-value-vectors-tokyo-apache-drill-meetup-20160322)などをごらんください）。
Spark 2.3では`ColumnVector`に関連するクラスを用いて、列データフォーマットのApache Arrowが持つデータを高速に読み書きできます。  
そして、この機能を利用して、従来からの性能上の懸念事項の１つであった[PySparkの実行速度の遅さ](https://www.slideshare.net/databricks/building-robust-etl-pipelines-with-apache-spark/40)、の改善が行われました。この遅さは、昔から指摘されていて、[PythonプロセスとJavaプロセスの間のデータフォーマット交換の非効率さ](https://codezine.jp/article/detail/8484)、によって引き起こされるものでした。この非効率さを解消するために、PythonとSparkの間のデータフォーマット交換自体をを無くしてしまいました。具体的には、Python側でpandasのAPIでアクセスされるデータをApache Arrow上に置き、そのデータをSpark側でも（内部的に）`ColumnVector`を使いApache Arrowを経由してアクセスしています（プルリクは[こちら](https://github.com/apache/spark/pull/18659)。その結果、3～100倍も性能が向上する、とDataBrickの[ブログ](https://databricks.com/blog/2017/10/30/introducing-vectorized-udfs-for-pyspark.html)でも取り上げられています。
使い方は簡単で、今までUDFの前で、`@udf("double")`と宣言していたのを、`@pandas_udf('double')`と書き直すだけです。先程のブログや[こちらのnotebook](https://databricks-prod-cloudfront.cloud.databricks.com/public/4027ec902e239c93eaaa8714f173bcfc/1281142885375883/2174302049319883/7729323681064935/latest.html)
にもプログラム例があります。
この成果については、まだSpark 2.3がリリースされていないにも関わらず、Spark Summit Europe 2017のKeynoteの３ページ目、で取り上げられました。

<iframe src="//www.slideshare.net/slideshow/embed_code/key/hrE8PNavKOeUDD?startSlide=3" width="595" height="485" frameborder="0" marginwidth="0" marginheight="0" scrolling="no" style="border:1px solid #CCC; border-width:1px; margin-bottom:5px; max-width: 100%;" allowfullscreen> </iframe> <div style="margin-bottom:5px"> <strong> <a href="//www.slideshare.net/databricks/deep-learning-and-streaming-in-apache-spark-2x-with-matei-zaharia-81232855" title="Deep Learning and Streaming in Apache Spark 2.x with Matei Zaharia" target="_blank">Deep Learning and Streaming in Apache Spark 2.x with Matei Zaharia</a> </strong> from <strong><a href="//www.slideshare.net/databricks" target="_blank">Databricks</a></strong> </div>

この機能に関しては、@ueshinさんと@BryanCutler(IBMのSpark Technology Center)の貢献が大変大きかったと思います。私も、初期段階にちょっとだけ実装の方針についてコメントしました。


### DataFrame/Dataset.cache()からの、データ読み出しの高速化

はじめに書いたように、Sparkではpandas上に置かれたデータを、Apache Parquetからデータを読み込んだ場合、`ColumnVector`を通してアクセス可能な列データフォーマットにデータを格納して、queryなどの処理を行う際に列データフォーマットから直接データを読み出して処理を行っています。  
さらに、`cache()`が実行されたときにも、インメモリキャッシュに列データフォーマットで保存されています。しかし、実装の歴史的な都合上、このデータは`ColumnVector`を経由してアクセスが行われていませんでした。さらに、列データフォーマットから一度行データフォーマットに変換してから、queryなどの処理が行われています。技術的には、このデータ変換は不要です。  
この
`cache()`の場合に、このデータ変換を取りさる[プルリクエストを私が1年9ヶ月前に投げた](https://github.com/apache/spark/pull/11956)後、いろいろな議論を経て[ようやくマージ](https://github.com/apache/spark/pull/18747)されました。簡単なベンチマークで、約3倍速くなっています。現時点ではスカラ変数の場合だけ有効で、[配列の場合](https://github.com/apache/spark/pull/19601)はまだ議論中です。


# あとがき

こんな改善も含まれるApache Spark 2.3ですが、いつリリースされるのでしょうか？  
[12月中頃に2.3用のブランチ作成、年明けにテストしてリリース](http://apache-spark-developers-list.1001551.n3.nabble.com/Timeline-for-Spark-2-3-td22793.html)、というのが11月頃の予定でしたが、12/20現在まだブランチは作成されていません。
