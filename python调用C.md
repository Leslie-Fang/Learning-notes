---
title: python调用C++
date: 2019-08-17 08:54:02
tags: 技术
---
## 介绍
Python调用C++时十分常见的方案，结合了Python的灵活以及C++的高效
一般的实现是将C++代码编译是动态链接库，提供Python到C++调用的API，然后将Python库打包成wheel包的形式发布

## 实现方案
常见的实现方案有boost.python, swig, ctypes, pybind11等
tensorflow采用的是swig的方案
boost.python为了兼容所有老的C++编译器，太重了
本文重点介绍pybind11

## pybind11的实现方案介绍
### setuptools定义wheel打包以及编译
使用setuptools去打包成wheel包，编写setup.py文件，定义setup函数
参考MLPerf的[setup.py文件](https://github.com/mlperf/inference/blob/master/loadgen/setup.py)
需要编译C++代码，setup函数中定义ext_modules字段指向一个Extension对象
Extension对象里面指定了需要编译的文件，编译宏以及依赖文件
之后借助setup.py我们可以进行打包，编译C++文件，生成wheel包
```
CFLAGS="-std=c++14 -O3" python setup.py bdist_wheel

##debug version
CFLAGS="-std=c++14 -O0 -g" python setup.py bdist_wheel
可以直接用gdb去debug
```

### Pybind11 python到C++的接口
参考[介绍](https://blog.csdn.net/fitzzhang/article/details/78988682) 以及[官网](https://pybind11.readthedocs.io/en/stable/index.html)
PYBIND11_MODULE 语法糖去定义一个python接口到C++接口的转换关系
