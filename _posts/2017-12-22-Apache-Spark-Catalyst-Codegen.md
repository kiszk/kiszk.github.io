---
layout: post
title: Apache Spark 2.3のCatalystでのコード生成の改善の解説
categories: ['ApacheSpark']
tags: ['spark']
published: true
---

# この記事は？
この記事は、[Distributed computing (Apache Hadoop, Spark, Kafka, ...) Advent Calendar 2017](https://qiita.com/advent-calendar/2017/distributed-computing)の21日目の記事です。

# この記事の内容は？
2018年の早い時期にリリース予定のApache Spark 2.3に入る、[Catalyst optimizer](https://www.slideshare.net/databricks/a-deep-dive-into-spark-sqls-catalyst-optimizer-with-yin-huai)によって生成されるJavaコードの改善に関するまとめです。

# 結局何がいいたいの？
- Spark 2.2までは、DataFrameやDatasetのqueryの中で、複雑な式や多数のカラム、を使うと、実行時に例外が投げられて、運が悪いと実行が止まってしまうのを、よく見ていたこと思います。
- この例外は、20年以上前に定義されたJavaのクラスファイルの仕様が持つ、２つの64KBの制限、からくるものでした。これらの制限によって起きる例外は、Spark 2.3ではかなり減ります。
- 64KBの制限を守ったとしても、OpenJDKの実装が持つ約8KBの制限、というものがあります。この制限にあたると、プログラムの実行が遅くなります。Spark 2.3では、この制限にあたりにくくする実装を盛り込んでいます。
- ただし、Whole-stage codegenでは、これらの問題がまだ一部残っているのでで、複雑な式・多数のカラム、があるプログラムには、whole-stage codegenによる最適化が適用されません。
- これらの作業の中で、マージされることなく消えていってしまったプルリクを、手厚く弔いたいと思います。


# 本論

## 複雑なqueryでみるエラーその１(64KBの壁との戦い１)

SparkのDataFrameやDatasetを使ってプログラムを書かれた方で、queryの中で複雑な式を書いたり、数千のカラムを持つデータに対してqueryを実行された方の中には、実行時にこのような例外を見て、実行が止まったり、実行時間が遅くなった、という経験をお持ちの方もいらっしゃるかと思います。

```
01:11:11.123 ERROR org.apache.spark.sql.catalyst.expressions.codegen.CodeGenerator: failed to compile: org.codehaus.janino.JaninoRuntimeException: Code of method "apply(Lorg/apache/spark/sql/catalyst/InternalRow;)Lorg/apache/spark/sql/catalyst/expressions/UnsafeRow;" of class "org.apache.spark.sql.catalyst.expressions.GeneratedClass$SpecificUnsafeProjection" grows beyond 64 KB
...
```

この例外は、過去にもJIRAでたくさん報告されて（[その1](https://issues.apache.org/jira/browse/SPARK-8443), [その2](https://issues.apache.org/jira/browse/SPARK-13242), [その3](https://issues.apache.org/jira/browse/SPARK-14793), [その4](https://issues.apache.org/jira/browse/SPARK-16191), [その5](https://issues.apache.org/jira/browse/SPARK-17092), [その6](https://issues.apache.org/jira/browse/SPARK-21720), [その7](https://issues.apache.org/jira/browse/SPARK-22226), まだ他にもあるはずです）、その都度fixされてきました

この問題は、DataFrameやDatasetを利用したプログラムからCatalystを用いて生成されたJavaのプログラムが、Javaバイトコードにコンパイルされた際、１メソッドのバイトコードの大きさは64KB以上であってはいけない、というJavaクラスファイルの制限に抵触してしまうために、発生する例外です。  
この制限による例外発生を避けるために、Spark内部では、コンパイルしてJavaバイトコードが64KB以上になりそうなメソッドは、CatalystがJavaプログラムを生成する際に複数メソッドに分割する、という解決策がとられてきました。

ある日、私がこのエラーに関して立て続けに修正のプルリク（
[その1](https://github.com/apache/spark/pull/19728), [その2](https://github.com/apache/spark/pull/19729), [その3](https://github.com/apache/spark/pull/19730), [その4](https://github.com/apache/spark/pull/19733), [その5](https://github.com/apache/spark/pull/19737)）を投げました。  
ちなみに、このプルリクを投げながら、こんなことを思っていましたが、反省しています...　その代わり、これからお話するようにかなりロバストになったと思います。

<blockquote class="twitter-tweet" data-lang="en"><p lang="ja" dir="ltr">4000 columnsなんてデータをSpark SQLに食わせるな、と思いながらPR書いたのだけど、以外に身近にユースケースがあったことを聞いて、反省している</p>&mdash; K. Ishizaki (@kiszk) <a href="https://twitter.com/kiszk/status/935846010256887808?ref_src=twsrc%5Etfw">November 29, 2017</a></blockquote>
<script async src="https://platform.twitter.com/widgets.js" charset="utf-8"></script>

これらのプルリクは無事にmergeされたのですが、この問題の多さに危機感を覚えたのか、コミッタの一人である@cloud-fanが根本的にこの問題を解決する意向を示しました。この問題にかかわる[topic JIRA](https://issues.apache.org/jira/browse/SPARK-22510)を作るとともに、複雑な式を使用した場合に統一してこの問題を解決する[プルリク](https://github.com/apache/spark/pull/19767)を作りマージしました。


## 複雑なqueryでみるエラーその２(64KBの壁との戦い２)

さらに、似ているけど微妙に違うエラーを見た方もいらっしゃるかもしれません。

```
Caused by: org.codehaus.janino.JaninoRuntimeException: Constant pool for class org.apache.spark.sql.catalyst.expressions.GeneratedClass$SpecificUnsafeProjection has grown past JVM limit of 0xFFFF
	at org.codehaus.janino.util.ClassFile.addToConstantPool(ClassFile.java:499)
	at org.codehaus.janino.util.ClassFile.addConstantNameAndTypeInfo(ClassFile.java:439)
	at org.codehaus.janino.util.ClassFile.addConstantMethodrefInfo(ClassFile.java:358)
...
```

これは、Catalystが生成したJavaプログラムがバイトコードにコンパイルされた際に、Javaクラスファイルが持つ別の64KBの制限に抵触してしまったことを示しています。Javaクラスファイルには、constant poolというクラス名、メソッド名、インスタンス変数名、文字列、などを格納する領域があります。このconstant poolのエントリ数は、64KB以上であってはいけない、というJavaクラスファイルの制限があります。  
この制限に抵触しないようにするために、

1. Catalystが生成したJavaファイルで、インスタンス変数をなるべく生成しないようにする[SPARK-22692](https://issues.apache.org/jira/browse/SPARK-22692)
2. どうしてもインスタンス変数を生成しないと行けない場合、スカラ変数ではなく配列要素として取るようにしてconstant poolの消費量を抑える([プルリク](https://github.com/apache/spark/pull/19811))

という試みが行われています。1.についてはほとんどマージが済み、2.もマージされました。ちなみに、このあたりのプルリクはコード上の似たような場所を変更するため、１つのプルリクがマージされるとまたrebase、ということでrebase祭り、なときもありました。  
（2.に関して補足すると、32768以上の配列要素をアクセスする際にも整数定数のロードにconstant poolエントリを消費するので、配列長は32768までに抑えています）


## 正しく動くことと速く動くこと、の違い(8KBの壁との戦い)

これらのプルリクによって、Spark 2.3では複雑なqueryや多数のカラムを持つDataFrame/Datasetを実行しても、例外で実行が止まってしまうことは、かなり減ったと思います。  
しかし、速度が改善しないことがあるかもしれません。それは、Java処理系が持つ機械語へのJust-in-timeコンパイラ（例えばOpenJDKではHotSpotコンパイラ）が、Javaバイトコードのコンパイルを行わないことがあるからです。例えば、OpenJDKでは8000byteまでのJavaバイトコードしか、機械語に変換しません（[OpenJ9](https://www.eclipse.org/openj9/)など、別のJava実装では別の制限になります）。これは、大きなJavaバイトコード列をコンパイルした結果、長大なコンパイル時間がかかるのを防ぐためかもしれません。  
そのため、あまり大きなJavaプログラムを生成してしまうと、結局インタープリタ実行になって実行速度は遅かった、ということになってしまいます。これを防ぐため、最初に述べた複数メソッドへの分割を行う際には、１メソッドが8000byteを超えない努力をしながら分割を行っています。

SparkのCatalystには、[whole-stage codegen](https://databricks.com/blog/2016/05/23/apache-spark-as-a-compiler-joining-a-billion-rows-per-second-on-a-laptop.html)という、複数のquery（例えば、複数の`filter()`）を１つのJavaメソッド内で処理する最適化されたJavaコードを生成する最適化があります。Whole-stage codegenでは複数のqueryを１つのメソッドで処理するようにするため、１つのメソッドのサイズが大きくなりがちです。  
実は、実装上の都合により複数メソッドに分割する機能は、whole-stage codegenというより最適化されたコード生成を行うパスでは動きません。従って、Whole-stage codegenでは、生成されるメソッドの大きさを抑制するため、下記のどちらかの条件に当てはまったときには、whole-stage codegenを止めて、１つ１つのqueryごとにJavaプログラムを生成していました。
1. 100より大きなフィールドを扱う文を持つqueryがある場合、whole-stage codegenを止める（オプション`spark.sql.codegen.maxFields`で決められています）
2. 8000byteを超えるJavaバイトコードが生成された場合は、whole-stage codegenを止める（[プルリク](https://github.com/apache/spark/pull/19083)）

１つ１つのqueryごとにコードを生成しても当然正しく動くのですが、whole-stage codegenで生成されたプログラムと比較すると、Javaプログラム間のデータ受け渡しのオーバヘッドが大きく、性能が今ひとつです。なんとかWhole-stage codegenが動くように、ということで、@maropuさんがコードが大きくなりがちなaggregationの場合に関して、whole-stage codegenでもメソッドを分割して生成できるようにする[プルリク](https://github.com/apache/spark/pull/19082)を投げました。  
コミッタの受けもよくマージされるかと思ったのですが、review中になんと

>「実装上の都合により複数メソッドに分割する機能は、Whole-stage codegenというより最適化されたコード生成を行うパスでは動きません」

という実装上の都合を取り払う、[画期的なプルリク](https://github.com/apache/spark/pull/19813)が別の人から投げられました。これがマージされ、@maropuさんのプルリクの機能をカバーしていたため、@maropuさんのプルリクはマージされることなくcloseとなり、闇に消えてしまうこととなってしまいました。  
ここで記録に残し手厚く弔いたいと思います...

さて、[画期的なプルリク](https://github.com/apache/spark/pull/19813)がマージされて一件落着。その効果は、@maropuさんが更新している、TPC-DSベンチマークで生成される[メソッドの最大のJavaバイトコードの大きさ](https://github.com/maropu/spark-tpcds-datagen/blob/master/reports/metrics/codegen-methods-max.csv)を、2017/12/12と2017/12/13で比較すると一目瞭然です。最大サイズが軒並み小さくなり、8000byteを超えるものはなくなりました。  
これで、めでたしめでたし、と思ったのですが、マージされた直後にデザイン上の問題が提起され、このプルリクは一旦revertされてしまいました。再マージにむけて[Design documentも用意され](https://issues.apache.org/jira/browse/SPARK-22600)、マージに向けた再議論がはじっています。将来的にマージされることとなるでしょう。しかし、リリースまでに残された時間も多くない中、Spark 2.3で入るかどうかはわかりません（個人的には、ぜひ2.3でマージされて欲しいプルリクです）。  
このプルリクがマージされれば、whole-stage codegenが適用されても１メソッドのJavaバイトコードが8000byteを超えることは、かなり減ります。さらに、[カラム数の制限を取り払うことができる](https://issues.apache.org/jira/browse/SPARK-22776)、はずです。


# あとがき

こんなドラマを持つApache Spark 2.3ですが、いつリリースされるのでしょうか？  
[12月中頃に2.3用のブランチ作成、年明けにテストしてリリース](http://apache-spark-developers-list.1001551.n3.nabble.com/Timeline-for-Spark-2-3-td22793.html)、というのが11月頃の予定でしたが、現在年明けすぐにbranchを作って一週間後にRC1をリリース、という提案が出ています。

実は、「かなり減ります」というところはもっと強く書きたかったのですが、修正しきれていないところを見つけてJIRAエントリ[SPARK-22868](https://issues.apache.org/jira/browse/SPARK-22868), [SPARK-22869](https://issues.apache.org/jira/browse/SPARK-22869)を作ったので、弱い表現になりました。
