---
title: tensorflow学习笔记
date: 2017-05-06 07:41:50
tags: 技术 tensorflow
---
## 前言
Tensorflow 是Google出的基于人工神经网络(ANN)的深度学习的框架，
[官网](http://example.net/)需要科学上网工具才能访问

## 安装
1. 为了避免python环境的污染，可以先安装一个python的虚拟环境VirtualEnv
2. 打开虚拟环境，用pip安装tensorflow的包
3. 和普通包一样使用tensorflow的包
4. 硬件有GPU别忘了使用CUDA

## 实验
源代码用的是[anishathalye](https://github.com/anishathalye/neural-style)大神的仓库
先看看最后训练出来的效果图
![tensorflow_result](http://omdthfazl.bkt.clouddn.com/tensorflow_result.jpeg "合成图")

原图是去年在上海迪士尼玩的时候随手拍的照片
![tensorflow_origin](http://omdthfazl.bkt.clouddn.com/tensorflow_origin.jpeg "原图")

想要训练的风格图片用的梵高的 **《星夜》**
![tensorflow_style](http://omdthfazl.bkt.clouddn.com/tensorflow_style.jpeg "星夜")

## 结论
<!--more-->
玩机器学习对计算机的硬件要求很高，这是GPU的强项
下面这个实验在一台cpu为i5-4200H，8G内存的台式机上跑的结果，整个训练过程时长了5个小时，运行过程中使用TOP命令查看，CPU一直处于100%以上的负荷，如果有GPU的话，理论上时间应该再20分钟左右
**结论** 没有GPU硬件的支持是玩不了机器学习的
