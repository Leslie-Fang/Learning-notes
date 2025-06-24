---
title: Jenkins
date: 2018-02-27 18:26:22
tags: 技术
---
## 介绍
Jenkins是一个自动化构建，测试工具 官网地址

## 教程
https://jenkins.io/doc/pipeline/tour/hello-world/#examples 启动服务器： java -jar jenkins.war --httpPort=8080

启动服务器之后在页面上配置代理： skip配置代理先 用户名:leslie 123456

进入之后看到用户管理页面 新建一个项目 在项目页面上构建build的命令 build之后可以在项目页面中，进入build结果的页面 查看build结果中命令行的输出

配置maven代理： 公司内网没有配置成功，还是无法连接上maven的中心仓库

## 和github的集成
可以在线安装插件
有时候代理不可用的时候也可以离线安装plugin https://stackoverflow.com/questions/14950408/how-to-install-a-plugin-in-jenkins-manually 插件下载地址： https://updates.jenkins-ci.org/download/plugins/git/
安装完插件之后如何和集成： https://www.jianshu.com/p/2836551a45ba
