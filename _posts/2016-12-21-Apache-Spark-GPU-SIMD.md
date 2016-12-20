---
layout: post
title: Spark.tcブログエントリ "Bringing Apache Spark Closer to SIMD and GPU"の解説
categories: ['ApacheSpark']
tags: ['spark']
published: true
---

# この記事は？
この記事は、[Distributed computing (Apache Hadoop, Spark, ...) Advent Calendar 2016](http://qiita.com/advent-calendar/2016/distributed-computing)の21日目の記事です。

# この記事の内容は？
12/16にSpark.tcに我々が投稿した[ブログ「Bringing Apache Spark Closer to SIMD and GPU」](http://www.spark.tc/simd-and-gpu/)の内容の解説です（元ブログは連載予定です）。

# 結局何がいいたいの？
* Project Tungstenの長期的なゴールの一つとして、SIMD/GPUなどの利用、があり、これを達成するためにいくつかの方法が考えられます
* ここでは、DataFrame/Datasetを用いて書かれたプログラムから、Java Just-In-Time (JIT)コンパイラがSIMD/GPUのコードを生成する、方法についてお話します
* DataFrame/Datasetを用いて書かれたSparkプログラムから内部的に生成される、現在のJavaコードと、JITコンパイラが効率的なSIMD/GPUのコードを生成可能なJavaコード、を比較してみて、必要となる最適化を考えてみます
* これらの最適化はプルリクとしてすでに投げていて、多くのものはSpark 2.0/2.1にマージされています。残りもSpark 2.2にマージされると信じています
* これらのプルリクは、SIMDを使わないCPUでも効果があり、マイクロベンチマークでは18倍以上高速化される場合があります

# Bringing Apache Spark Closer to SIMD and GPU
## 最初に

Apache Sparkは、コミュニティによる多くの貢献でによって、[目覚ましい性能向上](https://databricks.com/blog/2016/07/26/introducing-apache-spark-2-0.html)をとげています。
特に、最近のハードウェア・コンパイラの能力を引き出すための[Project Tungsten](https://databricks.com/blog/2015/04/28/project-tungsten-bringing-spark-closer-to-bare-metal.html)は性能向上の鍵となっています。

Project Tungstenの長期的なゴールの１つとして、SIMD/GPUなどのハードウェアの活用があります。この実現には、いくつかの方法が考えられます。

* SIMDやGPUに最適化されたコードをSparkライブラリとして提供する（例 [ALS](https://github.com/IBMSparkGPU/CUDA-MLlib/tree/master/als)）
* プログラマが書いたGPUカーネルを、Sparkの関数、例えば `map()`など、から呼ぶことができるようにする（例　[GPUEnabler](http://www.spark.tc/gpu-acceleration-on-apache-spark-2/)）
* GPUを使った他のライブラリを呼んだ結果をDataFrame経由でアクセスできるようなラッパーを用意する（例　[TensorFlow](https://databricks.com/blog/2016/01/25/deep-learning-with-apache-spark-and-tensorflow.html))
* DataFrameやDatasetを用いて書かれたプログラムから、内部的に生成されたJavaコードに対して、Java JITコンパイラがSIMD/GPUのコードを生成する。

この記事では、最後の方法について、現在の問題点とそれを解決する最適化についてお話します（詳細は連載にて）。

## この最適化って、SIMD/GPUにしか効かないんですか？
そんなことはないです。幸いにもSIMDを使わないCPUの場合にも効果があります。特に、Datasetで書かれた配列を扱うプログラムでは、18倍の性能向上を示しています。

## Project Tungstenについて

まず、Project Tungstenが行っていること（の一部）について簡単に説明します。  
下記のようなDataFrameを用いて書かれたプログラムについて考えてみます。

```scala
val df = sparkContext.parallelize(1 to 16, 1).map(i => (i.toLong, i * 2.5)).toDF("l", "d")
df.cache  // cacheを生成することを宣言
df.count  // cacheを強制的に生成する
val c = df.filter("l > 8").filter("d > 24").count
```

このプログラムは、内部的には下記の手順を経て、CPU上で実際に実行されます。

1. SQLレベルでの最適化を行います
1. DataFrame内の１行を読み込んで、複数の処理（例えば連続する`filter()`）を１つのループにまとめて、結果を書き出す、というループを持つJavaプログラムを内部的に実行します
1. このJavaプログラムがJava bytecodeに変換されます
1. Java bytecodeは、Java処理系が持つJITコンパイラによって、機械語命令にコンパイルされ、CPU上で実行されます

上記のプログラムから2.のステップで内部的に生成されるJavaプログラムのイメージが下記になります。  
このプログラムからは、SIMD/GPUのコードを生成するのはまだ難しそうです。例えば、whileループが並列可能かどうかを判断するのは（next rowの状態を知らないと）簡単ではありません。どのようなJavaプログラムが生成されると並列化解析しやすいか、次のセクションで考えてみましょう。

```java
int count = 0; 
while (next row is non-empty?) { // row内のデータは、Spark独自フォーマットで表現されています 
  read a row by calling getRow()
  get l from the row by calling getLong()
  if (l <= 8) continue;
  get d from the row by calling getDouble()
  if (d <= 24) continue;
  count++;
}
```

## JITコンパイラがSIMD/GPUのコードを生成するためには

JITコンパイラがSIMD/GPUのコードを生成するためには、下記のようなJavaコードがJITコンパイラの入力であるとうれしいです。

* ループは、`for (int i = START; i < END; i += STEP)` という形で、実行される範囲がわかっていて、並列化のための解析がしやすい
* column-orientedなデータ保存形式に対して、データの読み書きを直接行う
* ループ内の分岐命令が少ない
* call命令は無いか、あってもメソッドインライニング可能なもの

先程の例で言えば、下記のようなJavaプログラムが生成されると、JITコンパイラもSIMD/GPUのためのコードが生成しやすいです。  
forループになっていて、配列へのアクセスがループインデックスを用いています。これによって、並列化のための解析が容易になっています。

```java
long[] col0_l = {1, 2, ..., 16};
double[] col1_d = {2.5, 5.0, ..., 40.0};

int count = 0;
for (idx = 0; idx < 16; idx++) {  //  for loopが生成されています
  get e.l from col0_l[idx];  // column-orientedなデータ形式へのアクセス
  get e.d from col1_d[idx];  // column-orientedなデータ形式へのアクセス
  if (e.l <= 8) && (e.d <= 24) continue;
  count++;
}
```

## SIMD/GPUを利用するための道のり

実際にはまだ道のり半ばで、Spark側、JIT compiler側で、それぞれこんなことを改善していくことが必要だと考えています。

Sparkの改善点

* データ表現： column-orientedなデータ形式を（特に配列表現について）より効率的な表現にするとともに、データ生成を高速化する必要があります
* Javaコード生成： 並列化解析しやすいループを生成し、なるべく単純なコードをループ内に生成する必要があります
* Datasetに関するコード生成： Datasetの場合は、Spark内部のデータ表現とプログラマが書いたScalaのlambda式が使用するデータ表現の間のデータ変換が必要になり、できるだけ効率化する必要があります

JITコンパイラの改善点

* ループの並列化解析の拡張
* 無駄なコードの削除の拡張
* SIMD/GPUコード生成の拡張

Sparkの改善点は、すでにプルリクとして投げていて、下記の表にまとめました。多くのものはSpark 2.0/2.1にマージされていて、他のものもSpark 2.2にマージされると期待しています。

|変更の説明                               |JIRA |プルリク |状況  |性能向上  |
|:--------------------------------------|:----|:-------|:----|:--------|
|Parquetからのデータ読み出しのデータコピー削減|[13805](https://issues.apache.org/jira/browse/SPARK-13805)|[#11636](https://github.com/apache/spark/pull/11636)|2.0にマージ済|1.2倍|
|DataFrame/Dataset.cacheのデータの生成の高速化・読み出しのデータコピー削減|[14098](https://issues.apache.org/jira/browse/SPARK-14098)|[#15219](https://github.com/apache/spark/pull/15219)|2.2にマージ？|3.4倍|
|Spark内部の配列表現から間接参照を削減|[15962](https://issues.apache.org/jira/browse/SPARK-15962)|[#13680](https://github.com/apache/spark/pull/13680)|2.1にマージ済|1.7倍|
|配列を生成するDataSetプログラムから生成されるJavaプログラムが持つデータ変換オーバヘッドを削減|[17490](https://issues.apache.org/jira/browse/SPARK-17490)|[#15044](https://github.com/apache/spark/pull/17490)|2.1にマージ済|2.0倍|
|配列を読むDataSetプログラムから生成されるJavaプログラムが持つデータ変換オーバヘッドを削減|[15985](https://issues.apache.org/jira/browse/SPARK-15985)|[#13704](https://github.com/apache/spark/pull/15985)|2.1にマージ済|1.3倍|
|配列を生成するDataFrameプログラムから生成されるJavaプログラムが持つデータ変換オーバヘッドを削減|[16213](https://issues.apache.org/jira/browse/SPARK-16213)|[#13909](https://github.com/apache/spark/pull/16213)|2.2にマージ？|18.8倍|

JITコンパイラの改善点については、すでにオープンソースとして公開されている[Eclipse OMR](https://github.com/eclipse/omr)や将来オープンソースとして公開予定の[Open J9](https://www.infoq.com/news/2016/09/JavaOne-2016-IBM-Keynote-OpenJ9)にも、実装していくことを考えています。


# このエントリのあとがき

[Column-oriented storage](https://issues.apache.org/jira/browse/SPARK-15687)、[DataFrame/Datasetで配列を扱う際のオーバヘッド](https://issues.apache.org/jira/browse/SPARK-16070)、については、コミッタによってJIRAエントリがすでに立てられていますが、議論・実装が進んでいないのが現状です。  
この記事を読んで興味を持ちましたら、ご感想・気づいた点など、コメント・twitterなどでフィードバックをいただけるとうれしいです。さらに、JIRAへのコメント、（特に閑散とした）プルリクへのコメント、といった形で、ご意見をいただけると非常にうれしいです。

次回以降はデータ構造、実際に生成されているコード、を例にとって、プルリクによる変更を解説していきたいと思います。  
