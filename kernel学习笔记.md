---
title: kernel学习笔记
date: 2018-01-07 11:49:26
tags: 教程
---
* 用户态与内核态
http://jakielong.iteye.com/blog/771663

* linux启动过程
https://www.youtube.com/watch?v=yeMA7AJFtb8
https://www.youtube.com/watch?v=ALzySqflHJw
youtube:Basics of the Linux Boot Process

* 内存管理
http://gityuan.com/2015/10/30/kernel-memory/
youtube的教程：Virtual Memory，包括TLB的使用

* Hyper-threading 技术
Intel的CPU可以再BIOS仲打开hyper-threading
比如说2socket，每个socket有20个物理的core，打开hyper-threading之后可以有40个逻辑core，一共是80个逻辑core
可以通过
```
numactl --hardware
```
查看
socket 0上的逻辑core的编号是0-19,40-59
socket 1上的逻辑core的编号是20-39,60-79
可以通过
```
taskset -c 0,19(逻辑core的编号) process
```
将特定的process运行在这些逻辑core上面

* 进程与线程
进程通过调用fork这个kernel的api实现
线程通过调用Pthreads这个kernel的api实现
通过top可以看到进程的pid
通过top -p 进程pid可以看到这个进程包含的线程
