---
title: sed学习笔记
date: 2017-04-26 17:55:11
tags: 技术
---
# 学习使用sed
This blog is used to record the usage of sed.
sed 不会对原始文件产生影响，一行行读取，一行行匹配处理

## 正则表达式
正则表达式,在sed里面要加两个slash
* ^匹配开头  
* /^#/ 匹配#开头的行，注释行
* $匹配末尾
* /^$/ 匹配空行
* /./ 匹配一个字符
* /../ 匹配两个字符
* \* 匹配前面0个或多个字符

## 剔除指定行，打印剩余行
+ sed -e '1,5d' filename   删除1到5行，打印剩余行
+ sed -e '/^#/d' filename  删除所有#开头的行，打印剩余行
+ sed -e '/leslie/p' filename 删除所有包含leslie的行，打印剩余行

## 打印指定的行
* sed -n -e '1,5p' filename 打印1到5行
* sed -n -e '/^#/p' filename 打印所有#开头的行
* sed -n -e '/leslie/p' filename 打印所有包含leslie的行
* sed -n -e 'regureexpression1,regureexpression2' filename 从匹配第一个正则表达式的第一行开始到匹配第二正则表达式的第一行
* sed -e '=' filename    打印行号

<!-- more -->

## 替换
sed -e 's/word1/word2/g' filename
s表示替换，用word2替换word1
g表示全局替换，没有g则只会对第一个出现word1的位置替换为word2

## 组合多条命令
使用；分割多条命令
* sed -n -e '1,5=;1,5p' filename
或者
* sed -n -e '1,5=' -e '1,5p' filename

## 读取文本中的命令
新建一个 (command).sed后缀名的文件
在里面写入命令
sed -n -f command.sed filename
