---
title: HDFS
date: 2018-05-21 19:49:58
tags: 技术
---
## 介绍
使用bigdl的时候需要将训练数据集放在分布式文件系统中

## 准备环境
保证各个节点之间免ssh登陆
单节点搭建hadoop
https://hadoop.apache.org/docs/stable/hadoop-project-dist/hadoop-common/SingleCluster.html
多节点搭建hadoop

* namenode
系统环境变量：
export JAVA_HOME=/root/jdk1.8.0_161
export CLASSPATH=$JAVA_HOME/lib/dt.jar:$JAVA_HOME/lib/tools.jar:.
export HADOOP_PREFIX=/home/hadoop/hadoop-2.6.5
export HADOOP_HOME=$HADOOP_PREFIX/bin
export HADOOP_CONF_DIR=$HADOOP_PREFIX/etc/hadoop
PATH=$JAVA_HOME/bin:$JAVA_HOME/jre/bin:$PATH:$HOME/bin:/home/jiahaofa/spark/spark-2.2.1-bin-hadoop2.7/bin:$HADOOP_HOME:$HADOOP_PREFIX/sbin
export PATH

配置hadoop
etc/hadoop/hadoop-env.sh 写入 export JAVA_HOME=/usr/java/latest

配置etc/hadoop/slaves文件
r02s01
r02s02

配置etc/hadoop/core-site.xml
<configuration>
        <property>
                <name>fs.defaultFS</name>
                <value>hdfs://r02s01:9000</value>
        </property>
        <property>
                <name>io.file.buffer.size</name>
                <value>131072</value>
        </property>
        <property>
                <name>hadoop.tmp.dir</name>
                <value>file:///home/hadoop/hadoop-2.6.5/tmp</value>
        </property>
</configuration>

配置 etc/hadoop/hdfs-site.xml

<configuration>
        <property>
                <name>dfs.namenode.name.dir</name>
                <value>file:/home/hadoop/hadoop-2.6.5/tmp/dfs/name</value>
        </property>
        <property>
                <name>dfs.datanode.data.dir</name>
                <value>file:/home/hadoop/hadoop-2.6.5/tmp/dfs/data</value>
        </property>
</configuration>


* datanode
系统环境变量：
系统环境变量：
export JAVA_HOME=/root/jdk1.8.0_161
export CLASSPATH=$JAVA_HOME/lib/dt.jar:$JAVA_HOME/lib/tools.jar:.
export HADOOP_PREFIX=/home/hadoop/hadoop-2.6.5
export HADOOP_HOME=$HADOOP_PREFIX/bin
export HADOOP_CONF_DIR=$HADOOP_PREFIX/etc/hadoop
PATH=$JAVA_HOME/bin:$JAVA_HOME/jre/bin:$PATH:$HOME/bin:/home/jiahaofa/spark/spark-2.2.1-bin-hadoop2.7/bin:$HADOOP_HOME:$HADOOP_PREFIX:sbin
export PATH

配置hadoop
etc/hadoop/hadoop-env.sh 写入 export JAVA_HOME=/usr/java/latest

配置etc/hadoop/core-site.xml **一定要配置，很重要**
<configuration>
        <property>
                <name>fs.defaultFS</name>
                <value>hdfs://r02s01:9000</value>
        </property>
</configuration>

不配置 etc/hadoop/hdfs-site.xml


启动：
$HADOOP_PREFIX/bin/hdfs namenode -format
$HADOOP_PREFIX/sbin/start-dfs.sh

启动之后用jps查看namenode和datanode的进程

关闭：
$HADOOP_PREFIX/sbin/stop-dfs.sh

如果出现namenode或者datanode无法启动，查看机器上的log有明确信息

## hdfs命令测试
namenode:
hdfs dfs -put ./testdata/lesliename.txt /
其它节点
hdfs dfs -ls /
hdfs dfs -cat /lesliename.txt

datanode:
启动spark-shell：
val textFile = spark.read.textFile("hdfs://r02s01:9000/lesliename.txt")
textFile.first()


输出文件内容说明HDFS部署成功
