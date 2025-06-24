---
title: 同一OS下不同python2.7与3.4
date: 2017-10-15 21:39:17
tags: 技术
---
## 介绍
目前大部分程序都是基于python2.7开发的，但是python2只维护到2020年，未来是python3的，本文只要为了在一台机器上同时安装python2和python3，避免发生包冲突的问题。

## 安装
可以参考[链接1](http://joebergantine.com/articles/installing-python-2-and-python-3-alongside-each-ot/)
macos上默认安装的就是python2.7
直接
```
brew install python3
```
安装完成之后python -V
看到的还是python2.7.10

安装完成之后python3 -V
看到python3的版本

## 强烈建议使用virtualenv
[链接](http://www.marinamele.com/2014/07/install-python3-on-mac-os-x-and-use-virtualenv-and-virtualenvwrapper.html)
* 创建python2的虚拟环境
```
pip install virtualenv
python -V
virtualenv --no-site-packages venv3 --python=python2.7
```

* 创建python3的虚拟环境
```
pip3 install virtualenv
python3 -V
virtualenv --no-site-packages venv3 --python=python3.6
```
