---
title: MKLDNN与AVX512
date: 2019-01-19 17:02:02
tags: 技术
---
## 解释
以caffe2为例子进行解释：
caffe2的输入的数据格式为NCHW
batchsize，channel，height和width

1. 对于caffe2来说，在每一层卷积的时候就会直接把这个NCHW的数据塞进mkldnn对应的primitive里面进行计算，
2. mkldnn的primitive并行化计算包含两个层面，thread的层面和单个thread内部

### NH方向的并行化
对于NCHW的数据格式来说, N和H会对应了threads层面的并行化计算，也就是batchsize张图片或者一张图片很大也会被分配到多个不同的threads(每个thread可能绑定了CPU计算的core上)
**这解释了caffe2，detecture的时候batchsize从1增加到8，性能增加不明显的问题**
因为detecture的图片像素点很大1024*1024,正常resnet50的输入都是224*224，所以detecture的图片在H方向上已经在多threads上并行计算了，增大batchsize无法提高性能

### 为什么说c16（channel数是16的倍数）很重要
channel数是16的倍数，我们经常看到卷积核的输出的channel经常是32,64等16的倍数
#### AVX512
AVX512指令集的意思就是一次处理512bit的数据，一个FP32的数据占了32bit，所以一次性可以处理16个FP32的数据

对应了上面的卷积计算来说，假设有个NCHW的数据，一个batchsize有32个通道，那我们定义的卷积核也有32的通道
在第一个像素点上的卷积，可以先读取16个channel(通道)的第一个像素点的数据进来，对应了前16个卷积核的第一个像素点数据，进行一次卷积运算

同时，一个物理核(这里不确定是物理核还是逻辑核)会有 **16？？** 个AVX512可以读取的寄存器，每个寄存器可以存512bit，所以一次性可以读每个通道的前16个像素点进来，一共(16(通道数)*32(一个浮点数)*16(一组16个像素点组成的浮点数))一起存在16个AVX512的数据的寄存器里面
