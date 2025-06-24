---
title: numactl
date: 2017-12-02 15:26:43
tags: 技术
---
## 介绍
NUMA(Non-Uniform Memory Access)字面直译为“非一致性内存访问”
对于多核CPU而言，它的作用可以将程序绑定在固定的CPU的进程以及固定的内存块上，避免了程序在不同CPU和内存块之间切换导致的时间消耗。
从而可以提高程序的性能

## 查看cpu的配置
```
numactl --hardware
```
这里我们希望特定的程序运行在特定的memory以及特定的线程上面可以使用这样的命令
```
numactl -m 0(1) taskset -c 0,19 command to start the program
```


## auto numa balance
redhat系统默认打开auto numa balance
理论上使用numactl的时候就不会用到[auto numa balance](!https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/7/html/virtualization_tuning_and_optimization_guide/sect-virtualization_tuning_optimization_guide-numa-auto_numa_balancing)

## numastat
[numastat](!https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/7/html/performance_tuning_guide/sect-red_hat_enterprise_linux-performance_tuning_guide-performance_monitoring_tools-numastat)查看numa的使用情况
博客资料
http://dupengair.github.io/2016/10/12/%E7%B3%BB%E7%BB%9F%E6%80%A7%E8%83%BD%E6%B5%8B%E8%AF%95-%E7%B3%BB%E7%BB%9F%E6%80%A7%E8%83%BD%E5%B7%A5%E5%85%B7%E7%AF%87-numastat/
