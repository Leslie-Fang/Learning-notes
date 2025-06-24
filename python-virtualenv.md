---
title: python_virtualenv
date: 2017-10-15 21:19:49
tags: 技术
---
## 介绍
为了解决不同python程序之间，所依赖的包的版本不同导致程序无法运行的问题。

## 解决方法
解决方法可以是将在现有的程序代码的目录下运行
### 创建
创建一个venv目录，这个目录表示的就是一个python虚拟环境
```
virtualenv --no-site-packages venv
```
### 启动虚拟环境
一下命令可以启动虚拟环境
```
source venv/bin/activate
```

### 关闭虚拟环境
关闭虚拟环境
```
deactivate
```

### 删除
直接删除venv文件夹及可以

### 开发
在pycharm的default设置中可以选择到这个虚拟环境中的python，来解析pcharm中的包的依赖问题
