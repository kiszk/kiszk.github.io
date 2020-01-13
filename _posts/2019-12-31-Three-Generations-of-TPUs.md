---
layout: post
title: David Pattersonの講演の、TPUに関するざっくりまとめ
categories: ['TPU']
tags: ['TensorFlow', 'TPU']
published: true
---

# この記事は？

この記事は、[MLSE Advent Calendar 2019](https://qiita.com/advent-calendar/2019/mlse)の24日目の記事です。大変遅くなったことをお詫びいたします。

# この記事の内容は？
GoogleのDistinguished Engineerである[David Patterson](https://research.google/people/105290/)の、2019年10月にワシントン大学で行われた講演[Domain Specific Architectures for Deep Neural Networks: Three Generations of Tensor Processing Units (TPU's)](https://www.youtube.com/watch?v=VCScWh966u4)のTPUに関する部分のまとめです。  
TPUv3の概要が公開されたのはおそらく初めてだと思うので、機械学習システムの視点からTPU v2/v3の違いを知りたくてまとめてみました。

一刻も早く違いが知りたい、という人は講演中の下記のスライド に、TPU v1/v2/v3の比較がまとめられています（以下、全てのスライドはYouTubeからの引用）。
<br />
<!-- ![]({{ site.baseurl }}/images/post/2019-12-31/tpucomparison.png "TPU v1/v2/v3の比較"){: .img-mv} -->
<img src="{{ site.baseurl }}/images/post/2019-12-31/tpucomparison.png" width="50%" height="50%">


# 講演の内容について

（注）講演は以下の３つの話題を扱っていますが、このまとめでは最初のTPU、についてだけ触れます。
- TPU
- XLAコンパイラ
- ベンチマーク

この講演は、以下の２つのPaperが元になっています（２つ目はまだ出版されていないので、初めて聞く話もあるだろう、と話しています）。
- A Domain-Specific Architecture for Deep Neural Networks, Jouppi, Young, Patil, and Petterson, Communications of the ACM, September 2018
- A Domain-Specific Supercomputer for Training Deep Neural Networks, Jouppi, Yoon, Kurian, Li, Patil, Laudon, Young, and Patterson, Communications of the ACM, Summer 2020 (to appear)

## TPUの最初のストーリー（つまりTPU 1について）

2013年にdeep neural network(DNN)の新しいアプリの需要が爆発的に増えたので、DNNの推論のTotal Cost of Ownership(TCO)を10倍減らすことを目的にして、カスタムハードウェアを作ることにしました。非常に短い期間でTPUv1は開発されました。具体的には、2014年に始めて、15ヶ月後にデータセンターで使われていました。アーキテクチャ開発、ハードウェアデザイン、コンパイラ開発、テスト、デプロイ、全てをこの間に行いました。

<!-- ![]({{ site.baseurl }}/images/post/2019-12-31/tpuv1card.png "TPUv1のカードとパッケージング写真"){: .img-mv} -->
<img src="{{ site.baseurl }}/images/post/2019-12-31/tpuv1card.png" width="50%" height="50%">


### TPUv1のアーキテクチャ

700MHzのクロックで動作し、65536の8bit integer Multiply-acculate (MAC) unit、ます。
4MiBのon-chip accumulatorメモリ、24MiBのon-chip activationメモリ、を持ちます。
<br />
<!-- ![]({{ site.baseurl }}/images/post/2019-12-31/tpuv1architecture.png "TPUv1のアーキテクチャ図"){: .img-mv} -->
<img src="{{ site.baseurl }}/images/post/2019-12-31/tpuv1architecture.png" width="50%" height="50%">
<br />
<br />
activation bufferとMACは、性能に大きく関わる部分で、チップ面積の半分を占めます。
<br />
<!-- ![]({{ site.baseurl }}/images/post/2019-12-31/tpuv1floorplan.png "TPUv1のチップフロアプラン"){: .img-mv} -->
<img src="{{ site.baseurl }}/images/post/2019-12-31/tpuv1floorplan.png" width="50%" height="50%">
<br/>
<br/>
通常のプロセッサやGPUでは、MACでSRAMやレジスタにアクセスすることによって電力消費が発生します。これを減らすために、TPUでは[Systolic execution](https://apps.dtic.mil/dtic/tr/fulltext/u2/a066060.pdf)といわれる実行方法をとりました。性能/電力比は、対CPU(Haswell)で83倍、対GPU(K80)で29倍、改善されました。
<br />
<!-- ![]({{ site.baseurl }}/images/post/2019-12-31/tpuv1perfwatt.png "TPUv1の性能/電力比"){: .img-mv} -->
<img src="{{ site.baseurl }}/images/post/2019-12-31/tpuv1perfwatt.png" width="50%" height="50%">


### TPUv1の改善可能だった点

もしも、もっと開発時間があったとしたらどんなことをするべきだったか、というシミュレーションをしてみました。MXUを大きくする、クロックを上げる、速いメモリを使う、など。 
TPUv1ではメモリは標準のものを使ったので、それほど速くないです。もしDDR3 DRAMからGDDR5に変えることができたら(TPU')、メモリ帯域は34GB/sから180GB/sになります。すると、性能/電力比は、対CPU(Haswell)で196倍、対GPU(K80)で68倍、改善されるであろうという見込みがえられました。
<br />
<!-- ![]({{ site.baseurl }}/images/post/2019-12-31/tpuv1pperfwatt.png "TPU'の性能/電力比"){: .img-mv} -->
<img src="{{ site.baseurl }}/images/post/2019-12-31/tpuv1pperfwatt.png" width="50%" height="50%">

### TPUv1がうまくいった理由
以下の4つだと思います。
- 1次元ではなく、大きな2次元のMACユニットを用意したこと。
- 8-bit integerを採用したこと。
- Systolic arrayを採用したこと。
- キャッシュ、分岐予測といった、汎用CPU/GPUが持つ機能を採用しなかったこと。
<br />
<!-- ![]({{ site.baseurl }}/images/post/2019-12-31/tpuv1success.png "TPUv1がうまくいった理由"){: .img-mv} -->
<img src="{{ site.baseurl }}/images/post/2019-12-31/tpuv1success.png" width="50%" height="50%">


## TPUv2について
2014年に訓練(training)用として開発がはじまって、2017年にデータセンターにデプロイされました。

### 訓練はなぜ難しい？
以下の５つだと思います。
- より計算が必要であること。
- よりメモリが必要であること。
- よりプログラム可能であること。
- 桁数の多い数値を扱うこと。
- 並列化が易しくないこと。
<br />
<!-- ![]({{ site.baseurl }}/images/post/2019-12-31/trainingisharder.png "訓練のサポートが難しい理由"){: .img-mv} -->
<img src="{{ site.baseurl }}/images/post/2019-12-31/trainingisharder.png" width="50%" height="50%">
<br />
<br />
訓練は計算量が多い仕事なので、Machine learningにおいてブレークスルーを起こすためには、クラスタで構成される高速なSuper computerを作る必要があります。そのためには、ネットワークが重要です。  
<br />
TPUv2チップは、Inter-core interconnection(ICI)を4つ内蔵しています。
- 両方向で500Gbit/s
- 2Dトーラスを構成します。
- TPUv2チップを、wireを介して、ラック間であっても直接接続できます。
- ICIはTPUv2ダイの13%しか使っていません。
- 一般のデータセンターに比べて、5倍のバンド幅を、1/10のコストで実現しました。
<br />
<!-- ![]({{ site.baseurl }}/images/post/2019-12-31/tpuv2ici.png "TPUv2のICI"){: .img-mv} -->
<img src="{{ site.baseurl }}/images/post/2019-12-31/tpuv2ici.png" width="50%" height="50%">


### TPUv2のブロックダイヤグラム
- 128x128のMatrix Multiply Unit (MXU)
- 128x128のmatricesをTranpose, reduction, permuteするユニット(TRP)
- 32の2Dベクタレジスタと、2Dベクタメモリ(16Mib)を持つベクタユニット(VPU)。性能はMXUの1/10程度です。
- 1つのtensor coreあたり2つのHBM stackと64-bitバスで繋ぎました（トータルでTPUv1の20倍のバンド幅を確保しました。TPUv1でメモリバンド幅が足りなかった、という教訓を反映させました）。
<br />
<!-- ![]({{ site.baseurl }}/images/post/2019-12-31/tpuv2blockdiagram.png "TPUv2のブロックダイヤグラム"){: .img-mv} -->
<img src="{{ site.baseurl }}/images/post/2019-12-31/tpuv2blockdiagram.png" width="50%" height="50%">
<br />
<br />
フロアプランを見ると、TPUv1と違いTPUv2ではMXUはそれほど大きな部分を占めてはいません。
<br />
![]({{ site.baseurl }}/images/post/2019-12-31/tpuv2floorplan.png "TPUv2のフロアプラン"){: .img-mv}


### TPUv2の開発中に、neural network業界に何が起きたか？
2015年に[Batch normalization](https://arxiv.org/abs/1502.03167)、が登場！　Batch normalizationがneural networkの精度を改善して、訓練時間を1/14まで短縮しました。  
TPUv2では、ソフトとハードで対応しました。
- ソフトウェア： Batch normalizationを、バッチに対する足し算と掛け算に分解して、inverse square-rootを行います。
- ハードウェア： ベクタユニットのスループットを、最初のデザインより8倍にしました。inverse square root用のハードを追加しました。
<br />
<!-- ![]({{ site.baseurl }}/images/post/2019-12-31/batchnormalization.png "Batch normalizationのサポート"){: .img-mv} -->
<img src="{{ site.baseurl }}/images/post/2019-12-31/batchnormalization.png" width="50%" height="50%">


### TPU2のInstruction set architecture(ISA)
8演算まで実行可能な、322-bitのVLIW命令からなります。
- 2スカラarithmetic演算
- 2ベクタarithmetic演算
- 1ベクタロード
- 1ベクタストア
- 2つのMXUとTRPユニットとのやり取りのためのキュー
<br />
メモリ・レジスタは以下のようなものがあります。
64K 命令メモリ
32 32-bit スカラレジスタ
4K 32-bit スカラメモリ
32 128x8 32-bit ベクタレジスタ
32K 128 32-bit ベクタメモリ
<br />
<!-- ![]({{ site.baseurl }}/images/post/2019-12-31/tpuv2isa.png "TPUv2のISA"){: .img-mv} -->
<img src="{{ site.baseurl }}/images/post/2019-12-31/batchnormalization.png" width="50%" height="50%">

## TPUv3について
TPUv3は、TPUv2と同じテクノロジを使っています。違いは、
- クロックレート、ICIバンド幅、 HBMバンド幅、1.35倍に増加
- １チップで、コアあたりのMXUを1から2に増加
- ダイの大きさは6%しか増えていない
- 結果、消費電力1.6倍になったので、液冷を使用
- HBMのメモリ容量を２倍にして、コアあたり16GiBに
- システム全体では、256から1024チップに増加
<br />
<!-- ![]({{ site.baseurl }}/images/post/2019-12-31/tpuv3.png "TPUv3について"){: .img-mv} -->
<img src="{{ site.baseurl }}/images/post/2019-12-31/tpuv3.png" width="50%" height="50%">
<br />
<br />
<!-- ![]({{ site.baseurl }}/images/post/2019-12-31/tpusystempic.png "TPUv2とv3のラックの写真"){: .img-mv} -->
<img src="{{ site.baseurl }}/images/post/2019-12-31/tpusystempic.png" width="50%" height="50%">
<br />
TPUv1/v2/v3の比較表は、[この記事の最初](#この記事の内容は？)にあります。
