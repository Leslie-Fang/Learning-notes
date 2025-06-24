---
title: crontab
date: 2017-09-10 10:58:08
tags: 技术
---
## 介绍
crontab常用来周期性的执行命令。

## 启动

service cron(crond) status //ubuntu 和centos有不同

service cron(crond) start(restart,reload)  //如果没有启动，启动一下



## 用法

查看crontab命令：```crontab -l``` :系统级配置的任务好像无法通过crontab -l 来查看

编辑crontab命令:```crontab -e```

移除crontab命令:```crontab -r```

### 修改配置文件
系统级配置:在/etc/crontab 文件中编辑

用户级配置:用特定用户登录，在命令行中配置命令，是配置和用户相关的```crontab -e```

将命令写入文件中，然后直接crontab filename来读取文件中的所有命令

### 配置文件的写法：
* 周期性定时执行

分     小时    日       月       星期     命令

0-59   0-23   1-31   1-12     0-6     command     (取值范围,0表示周日一般一行对应一个任务)

日中有数字，表示的每个月到这一日都会执行
月中有数字，表示每年到这个月都会执行
星期中有数组，表示每周到这一天都会执行

* 在特定时候执行

@reboot command
such as：
@reboot root /bin/bash /test.sh


### debug
查看/var/log/cron文件
