---
title: MKL
date: 2018-01-19 17:06:48
tags: 技术
---
## 介绍
MKL是intel开发的一个数学运算库，集成在psxe软件包中。提供了一写API供开发者调用。

## 安装
首先安装psxe包含了所有的原件，或者单独安装mkl以及icc

## 如何使用MKL
[入门教程](https://software.intel.com/en-us/articles/intel-math-kernel-library-intel-mkl-2018-getting-started)

C语言：
在代码投include mkl的头文件：
#include "mkl.h"
在代码中调用mkl的API

## 编译：
source MKL的环境，主要配置环境变量，设置动态链接库的路径：
source /opt/intel/compilers_and_libraries_2018.1.163/linux/mkl/bin/mklvars.sh intel64

source一下icc
source /opt/intel/bin/iccvars.sh intel64

直接编译文件：
icc mkl-lab-solution.c -mkl
用makefile肯定也是类似的

生成一个可执行文件：
a.out
可以直接运行：
./a.out 1000
