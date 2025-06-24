---
title: caffe
date: 2017-11-11 16:01:57
tags: 技术
---
## 介绍
Caffe 是Berkeley Vision and Learning Center(BVLC)开发的一个深度学习的[框架](http://caffe.berkeleyvision.org/)，github[地址](https://github.com/BVLC/caffe),最初是用于图像处理，也可以用于自然语言处理。英特尔根据自己的CPU对caffe做了[优化](https://github.com/intel/caffe)，并发布了intelcaffe版本，调用MKL以及MKLDNN库，从而编译后使用最新的指令集加速运算。

## 基本概念
* [Blob](http://caffe.berkeleyvision.org/tutorial/net_layer_blob.html)就是caffe封装的数据，默认的输入图像的格式是(NCHW):number N x channel K x height H x width W
* solver: 定义一个solver文件,里面指定使用哪个net，步长多少

## 安装
* 安装一些系统依赖包
* 下载Intel caffe的源码
* cp Makefile.config.example Makefile.config
* 修改Makefile.config中的编译选项
* make -j 270 all（指定编译过程中可以使用的进程数量）

## 手写数字识别(Mnist)
### 准备数据
#### 下载数据
运行这个脚本
$CAFFE_ROOT/data/mnist/get_mnist.sh
下载数据集到当前目录下面$CAFFE_ROOT/data/mnist/
#### 转换数据格式
运行这个脚本去转换数据格式
$CAFFE_ROOT/examples/mnist/create_mnist.sh
会在$CAFFE_ROOT/examples/mnist/目录下面生成两个.lmdb后缀的文件，里面就是训练集和测试集
<!--more-->
#### LeNet模型
要使用caffe，最重要的就是定义模型，也就是编写对应的.prototxt文件
$CAFFE_ROOT/examples/mnist/lenet_train_test.prototxt

#### LeNet的求解器solver
solver定义了求解的参数
$CAFFE_ROOT/examples/mnist/lenet_solver.prototxt

#### 运行的环境变量设置与脚本
./examples/mnist/train_lenet.sh
在solver中有指定训练多少itertaion之后保存一次模型参数
可以用于后面进一步的训练或者inference
