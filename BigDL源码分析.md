---
title: BigDL源码分析
date: 2019-04-19 08:10:34
tags: 技术 深度学习 框架源码分析
---
## Install
### 先安装spark
https://spark.apache.org/downloads.html

### 编译bigdl
https://bigdl-project.github.io/master/#ScalaUserGuide/install-build-src/
```
bash make-dist.sh -P spark_2.x
```

## 运行的例子
每个model下面都有具体该如何运行的例子
https://github.com/intel-analytics/BigDL/tree/master/spark/dl/src/main/scala/com/intel/analytics/bigdl/models/lenet

训练的时候
java虚拟机的堆空间溢出
java.lang.OutOfMemoryError: Java heap space

设置driver的memory
```
spark-submit --driver-memory 12g --master local[1] --class com.intel.analytics.bigdl.models.vgg.Train dist/lib/bigdl-0.8.0-jar-with-dependencies.jar -f /home/dataset/cifar10/cifar-10-batches-bin -b 4
```

## Issue
 1. requirement failed: only mini-batch supported (2D tensor), got 1D tensor instead
 -b 指定的batchszie应该是core数量的4倍
https://github.com/intel-analytics/BigDL/issues/462

## 源代码阅读
如何在intellij中导入项目
https://www.jianshu.com/p/81a484ac83ae
http://dengfengli.com/blog/how-to-run-and-debug-spark-source-code-locally/


### 设计概念
1. 运行初始化时的模型构建
BigDL的设计是初始化的时候，一层一层将模型的读入，然后转化（每一层都寻找合适的BigDL的层，根据你输入的参数engine类型等），构建一个内部的图存在内存里面，以后的train或者inference的时候就使用这个内部的图

BigDL的INT8化也是这么做的，是在inference初始化的时候，读取FP32的模型，然后一层层转换成INT8的层，放到运行时内部的模型，正式inference的时候就使用这个内部模型

2. latency的问题
Spark适合进行批处理
但是针对一张图片进行inference的时候，因为spark需要构造图等初始化的工作，导致效率比较低
所以BigDL针对一张图片的latency的测试 **不使用spark**，走另外一套逻辑

3. MKLDNN的调用
通过JNI接口去调用C++ 代码
例子：
1. 模型中的调用ConvForwardDescInit
2. MklDnn.class 中的JNI函数
```
public static native long ConvForwardDescInit(int var0, int var1, long var2, long var4, long var6, long var8, int[] var10, int[] var11, int[] var12, int var13);
```
3. ConvForwardDescInit 是定义在子项目 BigDL-core里面
native-dnn/src/main/c/conv.c
```
JNIEXPORT long JNICALL Java_com_intel_analytics_bigdl_mkl_MklDnn_DilatedConvForwardDescInit(
  JNIEnv *env, jclass cls,
  int prop_kind,
  int alg_kind,
  long src_desc,
  long weights_desc,
  long bias_desc,
  long dst_desc,
  jintArray strides,
  jintArray dilates,
  jintArray padding_l,
  jintArray padding_r,
  int padding_kind)
```


### Resnet代码示例
看这个package：package com.intel.analytics.bigdl.models.resnet
目录里面的文件和结构：
ResNet 构建resnet的模型
DataSet
TrainCIFAR10这个例子


## 碎片知识
### 设置和读取环境变量
System.getProperty("leslie", "false")
可以在编译的时候-Dleslie=100去设定这个环境变量
在intellij里面为VM的参数
然后通过System.getProperty("leslie", "false")就可以去获取这个变量的值了，false这个地方填的是默认值

### this.synchronized 原子操作
https://www.jianshu.com/p/0cc4c330f25a
```
def getUniqueId() = this.synchronized {
    val freshUid = uidCount + 1
    uidCount = freshUid
    freshUid
  }
```
把这个函数定义成原子操作，也就是多个线程调用这个函数的时候，必须等一个线程调用结束之后，另一个线程才可以开始调用

## INT8
### scaling
```
spark-submit --master local[1] --driver-memory 50G --executor-memory 100G --executor-cores 32 --total-executor-cores 1 --class com.intel.analytics.bigdl.example.mkldnn.int8.GenerateInt8Scales ./dist/lib/bigdl-0.8.0-jar-with-dependencies.jar -f /home/dataset/bigdl_imagenet --batchSize 4 --model /home/lesliefang/bigdl/pre-trained-models/resnet-50.quantized.bigdl
```

### inference
```
spark-submit --master local[1] --driver-memory 50G --conf "spark.serializer=org.apache.spark.serializer.JavaSerializer" --conf "spark.network.timeout=1000000" --conf "spark.driver.extraJavaOptions=-Dbigdl.engineType=mkldnn -Dbigdl.mkldnn.fusion=true" --conf "spark.executor.extraJavaOptions=-Dbigdl.engineType=mkldnn -Dbigdl.mkldnn.fusion=true" --executor-memory 100G --executor-cores 1 --class com.intel.analytics.bigdl.example.mkldnn.int8.ImageNetInference ./dist/lib/bigdl-0.8.0-jar-with-dependencies.jar -f /home/dataset/bigdl_imagenet --batchSize 4 --model /home/lesliefang/bigdl/pre-trained-models/resnet-50.quantized.bigdl 2>&1 | tee test.log
```

## Lenet的例子
### 解决问题1：
Lenet 现在用了LeNet5 和 dnnGraph 两种方式，生成
```
} else {
  Engine.getEngineType() match {
    case MklBlas => LeNet5(10)
    case MklDnn => LeNet5.dnnGraph(param.batchSize / Engine.nodeNumber(), 10)
  }
}
```
这样导致mkldnn train后的模型也是用dnnGraph保存这会带来错误

现在新的BigDL的代码都已经改成
```
LeNet5(10)
```
用这个序列化的原始模型就可以了， MKLDNN 改写参考MKLDNN章节

### 解决问题2：
INT8化之前，模型需要保存成protubuf的格式而不是java的格式
解决方法：
java 的序列化方式是  model.save 和 Module.load，protobuf 对应的是 model.saveModule 和 Module.loadModule
Method 1: 在train的时候saveModel函数 用model.saveModule 而不是 model.save(推荐修改成这种method)
Method 2: 之前lenet train的时候模型是java序列化之后保存的，改下GenerateInt8Scales 用Module.load去载入模型

否则INT8化会报错
Exception in thread "main" java.lang.NullPointerException
        at com.intel.analytics.bigdl.utils.serializer.ModuleLoader$.initTensorStorage(ModuleLoader.scala:122)

### FP32 train
MKL
```
spark-submit --master local[26] --driver-class-path ./dist/lib/bigdl-0.8.0-jar-with-dependencies.jar --class com.intel.analytics.bigdl.models.lenet.Train ./dist/lib/bigdl-0.8.0-jar-with-dependencies.jar -f /home/dataset/mnist/type1 -b 104 --checkpoint /root/mnist_type1_trainsave/model
```
/root/mnist_type1_trainsave/model 目录下生成训练好的模型，取最新保存的一个作为test

### FP32 Test
MKL
```
spark-submit --master local[26] --class com.intel.analytics.bigdl.models.lenet.Test ./dist/lib/bigdl-0.8.0-jar-with-dependencies.jar -f /home/dataset/mnist/type1 -b 104 --model /root/mnist_type1_trainsave/model/20190813_000408/model.8656
```
Top1Accuracy is Accuracy(correct: 9854, count: 10000, accuracy: 0.9854)

MKLDNN
```
spark-submit --master local[26] --conf "spark.driver.extraJavaOptions=-Dbigdl.engineType=mkldnn" --class com.intel.analytics.bigdl.models.lenet.Test ./dist/lib/bigdl-0.8.0-jar-with-dependencies.jar -f /home/dataset/mnist/type1 -b 104 --model /root/mnist_type1_trainsave/model/20190813_000408/model.8656
```
Top1Accuracy is Accuracy(correct: 9854, count: 10000, accuracy: 0.9854)

### INT8 化
```
spark-submit --master local[1] --driver-memory 50G --executor-memory 100G --executor-cores 32 --total-executor-cores 1 --class com.intel.analytics.bigdl.example.mkldnn.int8.GenerateInt8Scales ./dist/lib/bigdl-0.8.0-jar-with-dependencies.jar -f /home/dataset/mnist/type1 --batchSize 4 --model /root/mnist_type1_trainsave/model/20190813_000408/model.8656
```

## MKLDNN 改写
### Train的时候
普通的sequential模型(不用DNNGraph)在DistriOptimizer第548行
```
val convertedModel = ConversionUtils.convert(model)
```
会去做原始OP向MKLDNN OP的转换的过程，但是和tensorflow不同的一点是，BigDL不会去插入reorder的节点，reorder的操作都是在运算过程中动态判断，动态生成的
生成的convertedModel模型，在第一次forward的计算的时候会去做MKLDNN的初始化操作

### Test的时候
Evaluator.scala 的test函数做MKLDNN化的改写
```
val modelBroad = ModelBroadcast[T]().broadcast(dataset.sparkContext,
  ConversionUtils.convert(model.evaluate()))
```
