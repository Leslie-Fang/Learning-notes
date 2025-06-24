---
title: tensorflow_dataset
date: 2020-04-24 20:35:54
tags: 技术 深度学习
---
## 介绍
tensorflow dataset是一个pipline，应该使用pipeline的思想去认识它
reference 资料:
* <<简单粗暴 TensorFlow 2.0>> 的第三章 https://tf.wiki/zh/basic/tools.html#tf-data
* 官方API：https://www.tensorflow.org/api_docs/python/tf/data
## 结构
示例代码 TF2.0：
```
fake_data = np.arange(1500)

dataset = tf.data.Dataset.from_tensor_slices(fake_data)

if is_mpi:
  dataset = dataset.shard(hvd.size(), hvd.rank())
dataset = dataset.shuffle(buffer_size=100)
dataset = dataset.repeat(num_epochs)
dataset = dataset.batch(batch_size=10)
dataset = dataset.map(parse_csv, num_parallel_calls=28)
dataset = dataset.prefetch(1)

for images, labels in dataset:
  images
  labels
```
tensorflow dataset 的使用代码可以分成三个阶段

### 阶段1：定义数据集的建立

* tf.data.Dataset.from_tensor_slices
直接从内存中的数据来创建dataset，数据全部在内存，适用于小数据集

* tf.data.TextLineDataset
传入文本文件名，创建的这个dataset时候，并不会将整个文件一起读到内存中

* tf.data.TFRecordDataset
传入tfrecord格式数据，

### 阶段2：数据集的转换
有很多转换的方法，根据需要我们可以选几种方法，推介使用这些方法的顺序：
* dataset.shard
利用horovod做分布式训练的时候，做数据并行化，shard数据集，每个train的实例分到一部分数据集来训练

* shuffle
设定一个buffersize的缓存区，取数据集前buffersize个数据放进来，每次要取数据的时候，从这个缓存区随机取batchsize的数据出来，然后在从dataset里面顺序补进来batchsize数据

buffersize的值设定很重要，否则可能起不到shuffle的作用

* repeat
repeat 几个epoch的数据，在后面真实在使用dataset的时候，
比如 estimator.train(input_fn=dataset) 或者 for images, labels in dataset:
如果repeat的epoch数据对应计算出来的我们可以train的steps 小于 指定train的steps
会产生tf.errors.OutOfRange，从而没有达到我们指定的steps就终止train
https://www.tensorflow.org/api_docs/python/tf/estimator/Estimator#train

* batch
将batchsize个样本组成一个batch数据，也可以叫做example

* map
设定我们原始dataset数据进行预处理的方法，返回一个新的dataset 也是(images,labels)元组的格式，num_parallel_calls指定预处理数据的时候可以使用多少个core
预处理不会立即执行，只有当我们访问dataset的iter去获取数据的时候才会执行

* prefetch
有个buffsize的参数，可以存buffersize个example到prefetch buffer里面，一个example是batchsize个样本
刚刚初始化的时候，就先取buffersize个example 执行map函数指定的预处理，将预处理之后的结果放入prefetch buffer里面

在访问一次iter的时候，会从这个prefetch buffer里面取一个example的数据(batchsize个样本)，这些数据送进去开始step的计算，同时后台执行map的预处理函数生成一个新的样本放入到prefetch buffer里面 **重要** 这两个操作是overlap的

### 数据的真实读取
* method1: 创建dataset_iter, 调用next(it)
调用一次，返回一个batchsize的数据

* method2: TF2.0:
```
for images, labels in dataset:
返回一个batchsize的数据
```
这里有一点很重要，如果设置了dataset.prefetch, 每一次遍历的时候只是从prefetch的buffer里面读取数据，不会阻塞的去做数据的预处理（执行dataset.map里面的函数），（下个step数据的预处理和这个step的计算同步做，填入到prefetch buffer）
如果没有设置prefetch，这里则会阻塞的去做数据的预处理，只有预处理好的数据才能读出来
