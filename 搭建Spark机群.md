---
title: 搭建Spark机群
date: 2018-03-01 18:59:38
tags: 技术
---
## 介绍
搭建一个spark机群进行数据分析
结构：主节点+2个工作节点
首先保证3个节点之间可以ssh免密码登录

## 依赖安装
* 主节点：
  * java
  * mvn：有必要配置代理,.m2/settings.xml
  * spark：用做命令行测试，开发java程序，用mvn下载依赖包
* 工作节点：
  * java
  * spark： 启动salve并连接主节点
正确安装各个依赖包之后，需要正确配置环境变量。

## 测试是否正确安装
### 主节点
* 测试spark命令行，[参考文档](http://spark.apache.org/docs/2.2.1/quick-start.html)
* 测试spark用mvn构建，[参考代码](https://github.com/Leslie-Fang/basicSpark)
```
mvn package
spark-submit --class "org.intel.dcg.leslie.SimpleApp" --master local[4] target/simple-project-1.0.jar
--class 指定jar包入口的class类
--master local[4] 运行在本地的4线程
```
### 工作节点
将主节点构建的jar包拷贝到工作节点
```
spark-submit --class "org.intel.dcg.leslie.SimpleApp" --master local[4] target/simple-project-1.0.jar
# --class 指定jar包入口的class类
# --master local[4] 运行在本地的4线程
```

## 构建机群
### 参考文档
* [工作原理](http://spark.apache.org/docs/2.2.1/cluster-overview.html#cluster-manager-types)
* [运行命令行](http://spark.apache.org/docs/2.2.1/submitting-applications.html)
* [standalone 主工具](http://spark.apache.org/docs/2.2.1/spark-standalone.html)
<!--more-->
### 步骤
1. 首先配置cluster managers，参考[standalone 主工具](http://spark.apache.org/docs/2.2.1/spark-standalone.html)
1.1 sbin/start-master.sh
通过查看log可以看到master的url
```
# start command
cd $spark_home/sbin
./start-master.sh
# read log
vim $spark_home/logs/spark-root-org.apache.spark.deploy.master.*.log
```
从log里面得到启动的master的url： spark://headnode:7077
这个变量要set到代码的SparkContext的master的变量里面
1.2 编写 conf/slaves file
```
vim $spark_home/conf/slaves
#content
knm009
knm010
```
1.3 在headnode上运行sbin/start-slaves.sh 运行所有slaves
在worknode的节点上查看slaves的所有的log以及java进程
```
cd $spark_home/sbin
./start-slaves.sh
```

2. 用[运行命令行](http://spark.apache.org/docs/2.2.1/submitting-applications.html)启动应用
```
# code:
https://github.com/Leslie-Fang/basicSpark
cluster branch
# build
mvn package
# how to run:
# 跑在worknodes上面
spark-submit --class "org.intel.dcg.leslie.SimpleApp" --master spark://headnode:7077 target/simple-project-1.0.jar
#　跑在本地
spark-submit --class "org.intel.dcg.leslie.SimpleApp" --master local[4] target/simple-project-1.0.jar
```

### 坑
问题：worknode的log里看到无法连接Failed to connect to master headnode:7077
http://blog.csdn.net/ybdesire/article/details/70666544
需要关闭headnode的防火墙
关闭之后将headnode上worknode上的spark进程都关闭
关闭之后，重启headnode上的spark-master以及spark-client

## 搭建 BigDL
BigDL是基于spark的进行DL的框架
[参考文档](https://bigdl-project.github.io/master/#ScalaUserGuide/install-build-src/)
### 安装
 ```
 git clone https://github.com/intel-analytics/BigDL.git
 cd $BigDL_Home
 # build
 bash make-dist.sh -P spark_2.x
 #生成一个dist的folder
 ```
### 使用
[入门](https://bigdl-project.github.io/master/#ScalaUserGuide/run/)
1. setup spark的cluster

2. 下载数据集合

3. 运行spark，指定bigDL的jar包
构建BigDL的jar包，和spark的multi-node一样的方式运行
```
spark-submit --master spark://headnode:7077 --executor-cores 1 --total-executor-cores 2 --class com.intel.analytics.bigdl.models.vgg.Train dist/lib/bigdl-0.3.0-SNAPSHOT-jar-with-dependencies.jar -f /home/lf/bigdl/data/cifar-10-batches-bin -b 2
```

### 坑
java虚拟机的堆空间溢出
java.lang.OutOfMemoryError: Java heap space
