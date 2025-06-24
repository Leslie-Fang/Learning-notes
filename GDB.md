---
title: GDB调试程序
date: 2017-09-25 19:10:55
tags: 技术
---
## 介绍
GDB可以用在linux shell下debug程序
GDB简明教程：
https://zhuanlan.zhihu.com/p/21987947
https://www.jianshu.com/p/08de5cef2de9

## 编译程序
在编译的时候手动加上-g选项
```
g++ -g file.cpp -o file
```

## 启动GDB环境
进入GDB调试环境
```
gdb file #file为编译后的可执行文件
```

## 查看源码
```
list #查看后10行
list - #查看前10行
help list #查看帮助，看到可以列出指定文件指定的行，或者函数
```
<!--more-->
## 添加断点
```
help break #查看帮助文档
b function_name
b row_num
b file_name:row_num
b row_num if condition

info break #查看断点
i b #查看断点，会返回断点的Num

disable 1 #禁用断点，Num 为info查看到的断点号

d 1 #删除断点，Num 为info查看到的断点号
```

## 运行程序
```
run(r)#运行程序
n #单步执行
until NUM #离开循环，执行到指定的行
step #跳入函数
finish #离开函数
quit #离开gdb环境
c #跑到下一个断点
```

## 打印变量
```
print var #打印变量
默认打印var长度为200个字符show print elements查看
设置无限制长度：set print elements 0

watch var #监控变量
info watch #查看监控的变量
whatis var #查看变量类型
x 虚拟地址 #查看特定虚拟地址上的值，默认一个值是4Bytes
x/1sb addr
https://blog.csdn.net/allenlinrui/article/details/5964046
```

## 打开图形化界面
```
wi #通过图形化界面查看debug过程更形象
```

## 查看调用栈
```
bt
```


## 具体列子
调试C++程序，想看C++在什么时候读取的图片
1. 直接在函数的位置打log

2. gdb加断点
写一个简单C函数打断点
extern "C" my_debug() {} //用C函数，编译后符号表里面函数名方便认识，方便用b 函数名打断点
my_debug() {print(\__line_)}
my_debug() ;
b my_debug
bt

gdb ./build/tools/caffe
set args time -model ./caffe_validation_models/TrueImageModels/lmdb/default_resnet_50/deploy.prototxt --forward_only --phase TEST --iterations 100 --engine=MKLDNN

3. 编译的时候去掉选项 -O2 -O3
添加编译选项-g

查看文件中是否有O3
find . -name "Makefile" | xargs grep O3
查找所有makefile文件
xargs命令的作用就是把每个查找到的文件送入grep

## 打印变量出错
gdb <optimized out>

因为编译时的优化，在makefile里面把O2去掉或者改成O0
或者makefile里面定义一个OPTIMIZATION变量OPTIMIZATION?=-O0
make OPTIMIZATION=-O0

## GUI工具
试了下gdbgui感觉还可以(https://www.gdbgui.com/)
```
pip install gdbgui

gdbgui #start
```
启动之后会自动弹出来服务器的firefox(如果服务器有桌面，工具配置了X11)，也可以关闭之后，像notebook一样在命令行配置
```
ssh -N -f -L localhost:5000:localhost:5000 root@remoteip
```
在本地chrome里面启动http://localhost:5000/
启动之后可以加载binary也可以attach process
右上角可以debug
右侧中边可以看到各个变量的值
