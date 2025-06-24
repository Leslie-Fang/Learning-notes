---
title: Spark
date: 2018-01-30 18:55:11
tags: 技术
---
## Install
* [Download](http://www.oracle.com/technetwork/java/javase/downloads/jdk8-downloads-2133151.html) & uncompressed jdk
* set the environment variables of java
```
export JAVA_HOME=/home/automation/java/jdk1.8.0_161
export CLASSPATH=$JAVA_HOME/lib/dt.jar:$JAVA_HOME/lib/tools.jar:.
export PATH=$JAVA_HOME/bin:$JAVA_HOME/jre/bin:$PATH
export SBT_HOME=/home/automation/spark/sbt/sbt
export SPARK_HOME=/home/automation/spark/spark-2.2.1-bin-hadoop2.7
export PATH=$PATH:$SBT_HOME/bin:$SPARK_HOME/bin
```
* [Download](https://www.apache.org/dyn/closer.lua/spark/spark-2.2.1/spark-2.2.1-bin-hadoop2.7.tgz) & uncompressed spark
* set the environment variables of spark
* install sbt： used to build scala

## Doc
[官方入门文档](https://spark.apache.org/docs/latest/quick-start.html)
* 打开命令行交互
dataset的构造函数中传入输入文件，调用dataset的[API处理文档]（https://spark.apache.org/docs/latest/api/scala/index.html#org.apache.spark.sql.Dataset）
* 编写可运行的文件
<!--more-->
build.sbt
SimpleApp.scala
放在规定的目录结构下面
```
import org.apache.spark.sql.SparkSession

object SimpleApp {
  def main(args: Array[String]) {
    val logFile = "YOUR_SPARK_HOME/README.md" // Should be some file on your system
    val spark = SparkSession.builder.appName("Simple Application").getOrCreate()
    val logData = spark.read.textFile(logFile).cache()
    val numAs = logData.filter(line => line.contains("a")).count()
    val numBs = logData.filter(line => line.contains("b")).count()
    println(s"Lines with a: $numAs, Lines with b: $numBs")
    spark.stop()
  }
}
```
**sbt package** 的时候如果有些包因为proxy无法下载
将包离线下载下来以后放到对应的目录下面 ~/.sbt/preloaded/***
