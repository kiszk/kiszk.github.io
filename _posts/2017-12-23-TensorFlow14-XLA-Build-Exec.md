---
layout: post
title: TensorFlow 1.4以降最新のマスターを用いた、ec2スポットインスタンスでのXLAのビルドと実行
categories: ['TensorFlow']
tags: ['TensorFlow', 'XLA', 'ec2']
published: true
---

# この記事は？

この記事は、[Data Platform Advent Calendar 2017](https://qiita.com/advent-calendar/2017/dataplatform)の21日目の記事です。

# この記事の内容は？

2017/12/22現在のTensorFlow 1.4+のmaster branchを使って、[XLA](https://qiita.com/GushiSnow/items/a373b8d5d904566f65fd)というTensorFlowプログラムの実行時コンパイラを有効にして、EC2上のインスタンスでビルドと実行する手順を説明しています。  
あらかじめ、CUDAとcuDNNがinstallされているAMIを使うので、CUDAとcuDNNのインストールなしに比較的楽にビルドできます。  
AMIで揃っている環境の都合で、CUDA 8とcuDNN 6を利用しています。


# TensorFlowプログラムをXLAで実行時にGPUのコードを生成して実行する、までの手順

ec2インスタンスの作成、環境設定、ビルド、実行、の順で説明していきます。

## ec2インスタンスの作成

`https://console.aws.amazon.com/ec2sp/v1/spot/home?region=us-east-1`から、スポットインスタンスを作成します。選択肢があるところは、下記のように選びました。
- バージニア北部で、p2.2xlargeで作りました(g2ではだめです)
- AMIは、community AMIsを`cuDNN`で検索して出てきた中で、CUDA8とcuDNN6を持つ、`RStudio-1.1.383_R-3.4.2_Julia-0.6.0_CUDA-8_cuDNN-6_ubuntu-16.04-LTS-64bit - ami-bca063c4`にしました。
- アベイラビリティーゾーンは、比較的安い値段が提示される us-east-1b/us-east-1c/us-east-1e にしました。
- デバイスは、/dev/sda1 16GB SSD、としました。
- 最高価格を、最近の相場の$0.35にしてみました

## ビルドのための環境設定

インスタンスにログインした後、面倒くさがりなので、全てルートで作業しました

ここのポイントは、bazelの最新版でもあまり古いものでもなく、bazel `0.7`を使うことです。

```
$ ssh -i YOUR_KEY.pem ubuntu@YOUR_IP
...
$ sudo bash
# apt-get update -y

# apt-get install -y nvidia-384
# export CUDA_HOME=/usr/local/cuda
# export LD_LIBRARY_PATH=$LD_LIBRARY_PATH:${CUDA_HOME}/lib64:${CUDA_HOME}/extras/CUPTI/lib64
# nvidia-smi  // この出力の中に "Tesla K80"とでていればOK
Fri Dec 22 17:45:20 2017
+-----------------------------------------------------------------------------+
| NVIDIA-SMI 384.90                 Driver Version: 384.90                    |
|-------------------------------+----------------------+----------------------+
| GPU  Name        Persistence-M| Bus-Id        Disp.A | Volatile Uncorr. ECC |
| Fan  Temp  Perf  Pwr:Usage/Cap|         Memory-Usage | GPU-Util  Compute M. |
|===============================+======================+======================|
|   0  Tesla K80           Off  | 00000000:00:1E.0 Off |                    0 |
| N/A   39C    P0    72W / 149W |      0MiB / 11439MiB |     99%      Default |
+-------------------------------+----------------------+----------------------+

+-----------------------------------------------------------------------------+
| Processes:                                                       GPU Memory |
|  GPU       PID   Type   Process name                             Usage      |
|=============================================================================|
|  No running processes found                                                 |
+-----------------------------------------------------------------------------+
# apt-get install -y openjdk-8-jdk gcc-4.9 g++-4.9 python3-numpy
# wget https://github.com/bazelbuild/bazel/releases/download/0.7.0/bazel-0.7.0-without-jdk-installer-linux-x86_64.sh
# chmod u+x bazel-0.7.0-without-jdk-installer-linux-x86_64.sh
# ./bazel-0.7.0-without-jdk-installer-linux-x86_64.sh
# bazel
# git clone https://github.com/tensorflow/tensorflow.git
# cd tensorflow
```
## ビルドのためのコンフィグレーション

一番悩むのがここかもしれません。Pythonは2でも3でもよいと思いますが、趣味でPython3にしました。
ポイントは、下記のところです。

- XLAは使う（当たり前）
- cuDNNのバージョンを聞かれた時に、`6.0.21`とマイナーバージョンまで入れる（`.so`ファイルの存在を探すため）
- clangは使わない
- インスタンスが提供するGPUのCuda compute capabilities（cc）は3.5以上（p2.xlargeなら3.5）であることを確認して、指定する（非XLA部分はcc3.0でも動くが、xlaのminimum supportは[cc3.5](https://github.com/tensorflow/tensorflow/blob/master/tensorflow/compiler/xla/service/platform_util.cc#L38)）
- gccは4.9を使う

`...[...]: `の後が実際に選んだ入力です。空白のところは、デフォルト値を使うため、単にEnterを押しました。

```
# ./configure
You have bazel 0.7.0 installed.
Please specify the location of python. [Default is /usr/bin/python]: /usr/bin/python3


Found possible Python library paths:
  /usr/local/lib/python3.5/dist-packages
  /usr/lib/python3/dist-packages
Please input the desired Python library path to use.  Default is [/usr/local/lib/python3.5/dist-packages]

Do you wish to build TensorFlow with jemalloc as malloc support? [Y/n]: y
jemalloc as malloc support will be enabled for TensorFlow.

Do you wish to build TensorFlow with Google Cloud Platform support? [Y/n]: n
No Google Cloud Platform support will be enabled for TensorFlow.

Do you wish to build TensorFlow with Hadoop File System support? [Y/n]: n
No Hadoop File System support will be enabled for TensorFlow.

Do you wish to build TensorFlow with Amazon S3 File System support? [Y/n]: n
No Amazon S3 File System support will be enabled for TensorFlow.

Do you wish to build TensorFlow with XLA JIT support? [y/N]: y
XLA JIT support will be enabled for TensorFlow.

Do you wish to build TensorFlow with GDR support? [y/N]: n
No GDR support will be enabled for TensorFlow.

Do you wish to build TensorFlow with VERBS support? [y/N]: n
No VERBS support will be enabled for TensorFlow.

Do you wish to build TensorFlow with OpenCL SYCL support? [y/N]: n
No OpenCL SYCL support will be enabled for TensorFlow.

Do you wish to build TensorFlow with CUDA support? [y/N]: y
CUDA support will be enabled for TensorFlow.

Please specify the CUDA SDK version you want to use, e.g. 7.0. [Leave empty to default to CUDA 9.0]: 8.0


Please specify the location where CUDA 8.0 toolkit is installed. Refer to README.md for more details. [Default is /usr/local/cuda]:


Please specify the cuDNN version you want to use. [Leave empty to default to cuDNN 7.0]: 6.0.21


Please specify the location where cuDNN 6.0.21 library is installed. Refer to README.md for more details. [Default is /usr/local/cuda]:


Please specify a list of comma-separated Cuda compute capabilities you want to build with.
You can find the compute capability of your device at: https://developer.nvidia.com/cuda-gpus.
Please note that each additional compute capability significantly increases your build time and binary size. [Default is: 3.7]


Do you want to use clang as CUDA compiler? [y/N]: n
nvcc will be used as CUDA compiler.

Please specify which gcc should be used by nvcc as the host compiler. [Default is /usr/bin/gcc]: /usr/bin/gcc-4.9


Do you wish to build TensorFlow with MPI support? [y/N]: n
No MPI support will be enabled for TensorFlow.

Please specify optimization flags to use during compilation when bazel option "--config=opt" is specified [Default is -march=native]: --config=opt


Add "--config=mkl" to your bazel command to build with MKL support.
Please note that MKL on MacOS or windows is still not supported.
If you would like to use a local MKL instead of downloading, please set the environment variable "TF_MKL_ROOT" every time before build.

Would you like to interactively configure ./WORKSPACE for Android builds? [y/N]: n
Not configuring the WORKSPACE for Android builds.

Configuration finished
```

## TensorFlowのビルド

ここまできたら、`bazel`におまかせ。ビルドには2時間強かかります。

```
# bazel build //tensorflow/tools/pip_package:build_pip_package
...
Target //tensorflow/tools/pip_package:build_pip_package up-to-date:
  bazel-bin/tensorflow/tools/pip_package/build_pip_package
INFO: Elapsed time: 8394.260s, Critical Path: 361.02s
# bazel-bin/tensorflow/tools/pip_package/build_pip_package /tmp/tensorflow_pkg
# virtualenv -p `which python3` --system-site-package $HOME/xla
# source $HOME/xla/bin/activate
(xla) # pip install /tmp/tensorflow_pkg/tensorflow-1.4.0-cp35-cp35m-linux_x86_64.whl 
```

もし、ソースファイルを更新して、リビルド＆再インストールしたい場合は、下記を実行すればOKです。
```
(xla) # bazel build //tensorflow/tools/pip_package:build_pip_package
(xla) # bazel-bin/tensorflow/tools/pip_package/build_pip_package /tmp/tensorflow_pkg
(xla) # pip install --ignore-installed --upgrade /tmp/tensorflow_pkg/tensorflow-1.4.0-cp35-cp35m-linux_x86_64.whl 
```

## XLAを使ってのTensorFlowの実行

パッケージに入っているサンプルを、[こちらのブログ](http://blog.adamrocker.com/2017/03/build-tensorflow-xla-compiler.html)を参考に実行してみましょう。実行すると、下記のような出力が画面に出ます。
```
(xla) # pushd ~/tensorflow/tensorflow/examples/tutorials/mnist/
(xla) # python3 mnist_softmax_xla.py --data_dir=./data
Successfully downloaded train-images-idx3-ubyte.gz 9912422 bytes.
Extracting ./data/train-images-idx3-ubyte.gz
Successfully downloaded train-labels-idx1-ubyte.gz 28881 bytes.
Extracting ./data/train-labels-idx1-ubyte.gz
Successfully downloaded t10k-images-idx3-ubyte.gz 1648877 bytes.
Extracting ./data/t10k-images-idx3-ubyte.gz
Successfully downloaded t10k-labels-idx1-ubyte.gz 4542 bytes.
Extracting ./data/t10k-labels-idx1-ubyte.gz
2017-12-22 23:57:19.793277: I tensorflow/core/platform/cpu_feature_guard.cc:137] Your CPU supports instructions that this TensorFlow binary was not compiled to use: SSE4.1 SSE4.2 AVX AVX2 FMA
2017-12-22 23:57:22.583086: I tensorflow/stream_executor/cuda/cuda_gpu_executor.cc:898] successful NUMA node read from SysFS had negative value (-1), but there must be at least one NUMA node, so returning NUMA node zero
2017-12-22 23:57:22.583476: I tensorflow/core/common_runtime/gpu/gpu_device.cc:1202] Found device 0 with properties:
name: Tesla K80 major: 3 minor: 7 memoryClockRate(GHz): 0.8235
pciBusID: 0000:00:1e.0
totalMemory: 11.17GiB freeMemory: 11.10GiB
2017-12-22 23:57:22.583507: I tensorflow/core/common_runtime/gpu/gpu_device.cc:1296] Adding visible gpu device 0
2017-12-22 23:57:22.828556: I tensorflow/core/common_runtime/gpu/gpu_device.cc:983] Creating TensorFlow device (/job:localhost/replica:0/task:0/device:GPU:0 with 10767 MB memory) -> physical GPU (device: 0, name: Tesla K80, pci bus id: 0000:00:1e.0, compute capability: 3.7)
2017-12-22 23:57:23.097159: I tensorflow/compiler/xla/service/service.cc:155] XLA service 0x7f7da40017b0 executing computations on platform CUDA. Devices:
2017-12-22 23:57:23.097214: I tensorflow/compiler/xla/service/service.cc:163]   StreamExecutor device (0): Tesla K80, Compute Capability 3.7
2017-12-22 23:57:24.509696: I tensorflow/stream_executor/dso_loader.cc:139] successfully opened CUDA library libcupti.so.8.0 locally
0.916
```

正常に実行が終了すると、最後にaccuracyが表示されて、実行時間の内訳を示す`timeline.ctf.json`ファイルが生成されます。そのファイルの中に、XLAでコンパイルされた命令列を示す`_XlaLaunch`という文字列が含まれていれば、正しくXLAで実行できています。

```
(xla) #　ls -alt
total 96
-rw-r--r-- 1 root root 20790 Dec 22 23:57 timeline.ctf.json
drwxr-xr-x 3 root root  4096 Dec 22 23:57 .
drwxr-xr-x 2 root root  4096 Dec 22 23:57 data
drwxr-xr-x 9 root root  4096 Dec 22 17:49 ..
-rw-r--r-- 1 root root  2776 Dec 22 17:49 BUILD
-rw-r--r-- 1 root root  9648 Dec 22 17:49 fully_connected_feed.py
-rw-r--r-- 1 root root   979 Dec 22 17:49 __init__.py
-rw-r--r-- 1 root root  1107 Dec 22 17:49 input_data.py
-rw-r--r-- 1 root root  6048 Dec 22 17:49 mnist_deep.py
-rw-r--r-- 1 root root  5187 Dec 22 17:49 mnist.py
-rw-r--r-- 1 root root  2683 Dec 22 17:49 mnist_softmax.py
-rw-r--r-- 1 root root  3631 Dec 22 17:49 mnist_softmax_xla.py
-rw-r--r-- 1 root root  8463 Dec 22 17:49 mnist_with_summaries.py
(xla) # grep _XlaLaunch timeline.ctf.json  | wc
     66     132    2278
(xla) # 
```

または、chromeで実行時間の内訳を見るとより分かりやすいかもしれません。  
下記のように自分のマシンで`scp`コマンドを実行して、`timeline.ctf.json`を自分のマシンに持ってきます。次に、chromeで`chrome://tracing/`をアドレスバーに入力して、`Load`ボタンを押すと、グラフが表示されます。その中に、`_XlaLaunch`という文字を含む横棒が表示されていれば、正しくXLAで実行できています。

```
 $ scp -i YOUR_KEY.pem ubuntu@YOUR_IP:/home/ubuntu/tensorflow/tensorflow/examples/tutorials/mnist/timeline.ctf.json .
```

# あとがき

今回は、AMIの都合でCUDA 8とcuDNN 6を使うことになりましたが、TensorFlow 1.4からCUDA 9とcuDNN 7が使えるので、そちらでのビルド手順もまとめたいとおもいます。
