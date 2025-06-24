---
title: MPI在高性能计算中的应用
date: 2017-10-07 18:19:41
tags: 技术
---
## 介绍
MPI(Message Passing Interface)在高性能计算中(High performance calculation 下面简称为HPC)具有广泛的应用。基本的应用范式可以分为两类:
* 单机多进程并行计算
* 多机集群并行计算

MPI的实现由以下几种库：
* MPICH
* open-mpi
* intel-mpi

其中前两个库是开源的，intel的库不是开源的，集成在intel的MKL当中

## MPI安装
可以通过两种方式安装：
1. OS的软件库安装：
  1. ubuntu 的apt-get install
  2. MAC-OS的brew install
  3. redhat的yum install
2. 通过源码包
比如下载mpich的[源码包](https://www.mpich.org/),安装过程就是一般的老三套:
  1. configure 添加编译选项生成makefile文件
  2. make编译
  3. make install 安装
  4. 如果有必要，将mpi/bin目录添加到环境变量

### Openmpi的安装
解压缩之后，编译安装https://www.open-mpi.org/faq/?category=building#easy-build:
```
wget https://download.open-mpi.org/release/open-mpi/v3.0/openmpi-3.0.4.tar.gz
./configure --prefix=/home/lesliefang/openmpi/openmpi_install/
make all install -j $logistic_corenum-1
PATH=$PATH:/home/lesliefang/openmpi/openmpi_install/
```

### 添加非root用户运行mpi
```
useradd multinode
```
各个节点的新用户都需要配置openmpi的信息

### 写hostfile
```
10.239.182.51
10.239.183.42
```
配置各个节点之间无密码ssh登录:要保证 **.ssh** 目录只有 **新添加的非root用户自己有权限** 否则配置的ssh不起作用，参考ssh免秘钥登录的文章
hostfile里面为各个节点的IP地址

关闭防火墙：
systemctl stop firewalld
systemctl disable firewalld

### 测试代码已经编译
```
#include "mpi.h"

int main(int argc, char *argv[])
{
 int nproc;
 int iproc;
 char proc_name[MPI_MAX_PROCESSOR_NAME];
 int nameLength;
 MPI_Init(&argc,&argv);
 MPI_Comm_size(MPI_COMM_WORLD,&nproc);
 MPI_Comm_rank(MPI_COMM_WORLD,&iproc);

 MPI_Get_processor_name(proc_name,&nameLength);
 printf("Hello World, Iam host %s with rank %d of %d\n", proc_name,iproc,nproc);

 MPI_Finalize();

 return 0;
}
```
```
mpicc -o hello hello.c
mpirun -N 2 -hostfile hostfile hello
```
-N参数指定的是单个节点上运行的进程数量
远程节点对应目录下面必须有相同的文件，否则这个远程节点不运行程序，不打印东西

## MPI 编程
基本的流程可以分成3个步骤
<!--more-->
1. 调用MPI的API，实现代码逻辑
2. 编译:mpic++ -std=c++11 -o 可执行文件名字 源码
3. 运行mpiexec -n 进程数 ./可执行文件 输入参数（mpiexec基本与mpirun等价）

## MPI的一般代码逻辑
<!--more-->
### 开启MPI:
```
MPI_Init(NULL, NULL);
MPI_Comm_size(MPI_COMM_WORLD, &comm_size);//comm_size is the number of process
MPI_Comm_rank(MPI_COMM_WORLD, &comm_rank);//comm_rank is the process's number
```
其中comm_size就是调用命令时传入的进程数量
comm_rank是每个进程的标识分别从(0到comm_size-1)

### 逻辑块
* 一般的编程思路是将大量独立的计算分割成几个块，每个块分别放入到一个进程中进行并行计算，计算结束之后，在第一个进程(rank 0)中搜集各个进程的计算结果，归总判断，进行下一轮计算或者结束计算
* 在编程过程中可以通过comm_rank来判断当前处在哪个进程中
* 通过调用MPI_Barrier(comm)，其中comm为一个对象承担各个进程之间通信的任务，则会等待所有进程都执行到这一句代码，每个进程才会继续往下执行
* 数据的搜集:在每一轮计算之后，搜集各个进程的计算结果到rank 0中
```
if(my_rank != 0){
    MPI_Send(&has_change, 1, MPI_BYTE, 0, 0, comm);
}else{
    bool localhas_change = false;
    for(int j=1;j<p;j++){
        MPI_Recv(&localhas_change, 1, MPI_BYTE, j, 0, comm, &state);
        has_change = localhas_change | has_change;
    }
}
```
  每个进程都把各自的数据发出来，进程0搜集到每个进程发出来的数据之后，对数据进行逻辑运算
* 数据的广播:主要用于数据的同步，可以用在计算开始之前以及每一轮计算结束并搜集数据准备开始下一轮计算之前
```
if(my_rank == 0){
    for(int k = 1; k < p; k++){
        MPI_Send(&has_change, 1, MPI_BYTE, k, 0, comm);
    }
}else{
    MPI_Recv(&has_change, 1, MPI_BYTE, 0, 0, comm, &state);
}
```
  进程0将归纳好的数据发出来，其它进程搜集到这些数据之后进行数据的同步

### 结束MPI
当计算结束之后，结束MPI
```
MPI_Finalize();
```

## 实例代码
可以参考我的[github仓库](https://github.com/Leslie-Fang/MPI)，这个仓库中的serial_bellman_ford.cpp以及mpi_bellman_ford.cpp分别通过串行以及mpi并行的方式实现了[Bellman-Ford算法](http://www.wutianqi.com/?p=1912)计算单源图的最短路径。
