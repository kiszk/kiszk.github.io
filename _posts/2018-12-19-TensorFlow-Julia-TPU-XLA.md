---
layout: post
title: JuliaからCloud TPUを使う論文の、ざっくりまとめ
categories: ['TensorFlow']
tags: ['TensorFlow', 'TPU', 'XLA', 'Julia']
published: true
---


# この記事は？

この記事は、[TensorFlow Advent Calendar 2018](https://qiita.com/advent-calendar/2018/tensorflow)の17日目の記事です。

# この記事の内容は？

Arxivに投稿された論文、[Automatic Full Compilation of JULIA Programs and ML Models to Cloud TPUs](https://arxiv.org/abs/1810.09868)のまとめです。特に、TPUのためのコードを生成するためにはどうすればよいか、ということについてまとめます。一言でまとめれば、
```
最近のTensorFlowだったら、XLA用の中間コード生成してコンパイルすればTPU使えるよ
```
です。この論文に書かれている内容は、[XLA.ji](https://github.com/JuliaTPU/XLA.jl)で試すことができるようです。ただし、残念ながら論文に書かれている部分はコンパイル済みバイナリとして提供されていて、実装を見ることはできません。

# とにかく、JuliaのコードをTPUで動かしてみたい人向け

論より証拠、という方は[XLA.ji](https://github.com/JuliaTPU/XLA.jl)として公開が実装されているので、こちらを実行してみてください。詳しい使い方は、[XLA.jl を試してみた](https://qiita.com/antimon2/items/ccfb5c2353d99fcb1976)、[Julia at NIPS and the Future of Machine Learning Tools](https://juliacomputing.com/blog/2018/11/15/julia-ml-three-papers.html)、を読むとわかります。  

# 論文の内容について

## 概要

この論文では、新しいAPIとGoogle XLAコンパイラを使って、Juliaプログラムのある部分をTPUにオフロードする方法を提案しています。実際に、Juliaで書かれたVGG19のforward passを、完全に融合(fuse)して、単一のTPU executableに変換できます。すでに提案されている自動微分の手法を使って、VGG19のbackward passを自動生成してTPUで実行できます。  
実行性能に関して、VGG19のforward passを100枚の画像について実行したとき、CPUで52.4秒かかったのが、TPUでは0.23秒ですみます。この実装はJuliaで1000行以下で、Juliaコンパイラやライブラリに対してTPUに特有の変更は行っていません。


## １章：はじめに

2018年9月に、Googleは、XLA("Accelerated Linear Algebra")コンパイラのlow level IR(intermediate representattion、中間表現)を通して、TPUを使えるようにしました。このIRは汎用的であり、線形代数の基本的な演算を最適化するためにコンパイラで使われます。このIRは、TensorFlow以外のユーザによって、TPUを使うためのいい道具です。

Tensorflowでは、XLAに入力される計算グラフをPythonプログラムを実行しながら作っています。一方、我々のアプローチでは、Julia programを実行前のコンパイル時に解析しながら計算グラフを作ります。この方法によって、Julia言語の高い表現力を完全に利用することができます。例えば、1) multi dispatch、 2) high order functions、 3)  微分方程式のソルバ、 4) 汎用的な線形代数ルーチン、などです。さらに、[`Zygote.jl`](https://github.com/FluxML/Zygote.jl)を使って、自動微分も行います。

## ２章：TPUのシステムアーキテクチャについて

### TPUハードウェア

Google CloudのTPU v2は、4つのTPUv2チップ（つまり8 TPUv2コア）を提供しています。このTPUは、シリアライズされたXLA IRをウエブサービス経由で受け付けます。XRTと呼ぶこのAPI（[こちら](https://blogs.yahoo.co.jp/verification_engineer/71768836.html)のアドベントカレンダーでも触れられています）によって、TensorFlow以外のユーザがXLA IRを生成することができます。XRTは、2018/9/27にTensorFlow 1.11がCloud TPUにデプロイされたときから使えるようになりました。

### XLA

XLA("Accelerated Linear Algebra")は、Googleによる"partially"なオープンソースコンパイラプロジェクトです。バックエンドには、CPU、GPU、TPUがあります。HLO("High-Level Optimization") IRが、XLAの入力となり、基本的演算・線形代数演算・配列演算を、bloat16などを含むスカラ型の配列・タプル（タプルの配列は受け付けない）に対して行います。XLAは、演算・使用されるメモリ量の最適化、なども行います。  
HLOはprotobufによってシリアライズ可能で、XRTが受け付けることができるフォーマットです。  
HLOの命令には２種類あります。

1. Static operation：すべての値がコンパイル時に分かっていないといけない
1. Dynamic operation：テンソルの形やレイアウトがわかっていれば、大きさなどはコンパイル時に分からくてもよい

## ３章：Juliaコンパイラについて

Julia言語はdynamic languageですが、コンパイラバックエンドとして静的型付けなコンパイラであるLLVMを使っています。Juliaコンパイラは、このギャップを埋めるために、下記の４つのことを行っています（このまとめでは詳細は省きます）。

1. 動的なセマンティクス
1. 静的コンパイラ用の中間表現への変換
1. メソッドをまたいだ型推論
1. 静的な範囲の解析

## ４章：XLAを使うために

ここでは、XLAに対応するために行ったことを説明します。

### Tensor表現の扱い

配列に関して、XRT clientから渡される配列とTPU内で扱われる配列を１つにして、XLAで必要な情報を扱うために`XRTArray`という構造体を、Juliaで新たに定義しました。

### 命令の表現

HLOがもつ、static operationとdynamic operationを、Julia上で扱えるように、新たに構造体と関数を定義しました。

### オペランドの形(shape)について

HLOの各命令のオペランドの形(shape)は、型推論で決定できるはずです。

## ５章：JuliaのセマンティクスをXLAに変換するために

主に以下の２つが必要になります

1. 構造体のマッピング
1. 制御構造（control flow）の扱い

## ６章：推論について

このコンパイラが行う推論は、かなり大変なもので、コンパイラの推論部分にかなりの負担をかけます。  
我々の実装は、今のJuliaコンパイラの推論の実装の一部を変更しています（バグフィックスや、オプションで推論の一部を制御する）。Julia communityにfeedbackする予定です。

## ７章：結果

ここでは、プログラムの変換結果と、実行性能を示します。

### 単純な例による変換

VGG19の一部であるこんな単純なプログラムは、
```
dense(W, x, b) = (W * x) .+ b
softmax(xs) = exp.(xs) ./ sum(exp.(xs))
```
このような、XRTの入力であるXLA IRに変換されます
```
@code_xla opt=true dense(W, x, b)
c1 {
  c1p0 = f32[] paramater(0)
  c1p1 = f32[] paramater(1)
  ROOT c1a2 = f32[] add(c1p0, c1p1)

  ENTRY dense 
    c0p0 = f32[10,10]{0, 1} paramater(0)
    c0p1 = f32[10]{0, 1} paramater(1)
    c0d3 = f32[10]{0} dot(c0p0, c0p1),
      lhs_contracting_dims={1},
      rhs_contracting_dims={0}
    c0p2 = f32[10]{0} parameter(2)
    ROOT c0m4 = f32[10]{0} map(c0d3,
      c0p2), dimensions={0}, to_apply=c1
}
```

### VGG19のforward passとbackward pass

我々が使ったVGG19の実装は[Metalhead package](https://github.com/FluxML/Metalhead.jl)で、[FluxML framework](https://github.com/FluxML/Flux.jl)によって線形代数演算に変換されます。我々のコンパイラは、このforward passの変換結果を完全に推論して最適化し、１つの実行モジュールにコンパイルできました。  

Backward passは、[`Zygote.jl`](https://github.com/FluxML/Zygote.jl)で変換を行います。この変換結果は、forward passより難しいコードですが、コンパイラを改良することで、他のXLA frontendのようなXLA IRを生成することに成功して、TPU用のコードを生成できました。


### 実行性能

現時点の結果であって、XLAを改良すればまだ性能は上がるかもしれません。  
バッチサイズ100のとき、CPUで52.4秒かかっていたのが、TPUでは（計算部分だけ見れば）0.23秒になりました。

|N|1|10|100|
|:---|:---|:---|:---|
|Flux CPU|0.79s|6.67s|52.4s|
|PyTorch CPU|1.16s|9.55s|93.0s|
|FluXLA CPU|12.06s|64.8s|>600s|
|FluXLA TPU (total)|0.86s|0.74s|0.93s|
|FluXLA TPU (compute)|0.02s|0.04s|0.23s|

N: バッチサイズ
CPU: 20-cores Intel Xeon Sliver 4114 @ 2.2GHz with AVX512

Flux CPU: FluxとJuliaのmaster branch
Pytorch CPU: Pytorch
FluXLA CPU: FluXLA(この論文の実装)で、XRT経由でCPU上で実行
FluXLA TPU (total): FluXLA(この論文の実装)で、XRT経由でTPU上で実行
FluXLA TPU (compute): FluXLA(この論文の実装)で、XRT経由でTPU上で実行したときのTPUの実行時間

## ８章：現在の制限と今後について

現在のコンパイルモデルは、"all or nothing"で領域内にXLA IRに変換できない操作が１つでも入っていると、領域全体が変換に失敗します。これは、現在改良中です。デバッグしにくさも、現在の問題の一つです。  
XRTArrayの要素の型が、XLAがサポートしているものに限られています。これによって、レイアウトの変換にも制限が出てしまっています。

## 謝辞

論文を参照してください
