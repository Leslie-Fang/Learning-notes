---
title: svn
date: 2018-08-30 07:35:41
tags: 技术
---
## 一般先在服务器上建立个仓库
```
mkdir /root/CgiNg
svnadmin create /root/CgiNg
```
## 进入/root/CgiNg修改conf相关配置文件
一般相关的有三个配置文件
authz：负责账号权限的管理，控制账号是否有读写权限
passwd：负责账号和密码的用户名单管理
svnserve.conf：svn服务器配置文件
具体修改哪些内容，可以参考这个[链接](https://www.cnblogs.com/mymelon/p/5483215.html)

## 启动服务
```
svnserve -d -r /root/CgiNg
```

## 下载仓库
windows上用tortoise svn
右键checkout，输入地址svn://149.28.149.94:3690，和下载到本地的地址
Linux上，
```
mkdir repo && cd repo
svn co svn://149.28.149.94:3690
```
<!--more-->
## 提交文件
* 所有文件
```
svn add ./*
svn commit -m "message"
```
注意在服务器端仓库中文件是用另外格式存储的，所以你在服务器端find是找不到代码文件的
* 单独文件
```
svn add 文件名
svn commit -m "message"
```
* 目录以及目录下文件
```
svn add 目录
svn commit -m "message"
```

## 删除文件(注意删除本地和远程的)
两种方法：
* 方法1直接删除远程文件
svn　delete　svn://路径(目录或文件的全路径) -m “删除备注信息文本” 推荐如下操作：
* 先删除本地文件然后commit上去
svn　delete　文件名
svn　ci　-m　“删除备注信息文本” <- ci是commit的简写

## 不小心add了多余的文件
```
svn add ./*
之后发现有的文件不应该提交
svn revert 文件
svn revert --depth infinity 目录
```

Linux端命令[详细手册](https://vosamo.github.io/2015/11/Linux-SVN/)
