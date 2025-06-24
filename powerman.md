---
title: powerman
date: 2017-06-10 15:56:35
tags: 技术
---
## 介绍
powerman是一个开源的工具，被谷歌用于cluster的机群管理
通过SNMP协议可以管理PDU电源
通过ipmi协议可以管理机器的reboot以及开关机

## 安装
源码下载[地址](https://github.com/chaos/powerman)
安装流程参考源码中的INSTALL文件
1. configure自动生成makefile
```
./configure
```

2. 编译
```
make
make check //check the make result
```

3. 安装
```
make install
```

<!--more-->
