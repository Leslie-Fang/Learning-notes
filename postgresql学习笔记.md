---
title: postgresql学习笔记
date: 2017-10-15 16:02:57
tags: 技术
---
## 介绍
PostgreSQL相对于竞争者的主要优势为可编程性：对于使用数据库资料的实际应用，PostgreSQL让开发与使用变得更简单。
微信支付核心数据库也是基于 PostgreSQL[链接](https://www.qcloud.com/community/article/164816001481011854)
## 安装
ubuntu下面:
* 安装客户端
sudo apt-get install postgresql-client
* 安装服务器
sudo apt-get install postgresql
安装完成后，PostgreSQL服务器会自动在5432端口开启，
所以如果在阿里云上部署，需要远程访问的话记得在阿里云管理界面上管理规则组中打开5432端口

## 创建新用户和数据库
安装postgresql之后默认生成一个名为postgres的数据库和一个名为postgres的数据库用户，同时还生成了一个名为postgres的Linux系统用户，有两种[方法](http://www.ruanyifeng.com/blog/2013/12/getting_started_with_postgresql.html)
使用shell命令行的方法
### 创建数据库用户
创建数据库用户leslie，并指定其为超级用户
```
sudo -u postgres createuser --superuser leslie
```
如果原来是在root用户下
```
su - postgres
createuser --superuser leslie_postgres3
```

### 登录数据库控制台,设置账号密码
psql命令登录PostgreSQL控制台
```
sudo -u postgres psql  #login the PostgreSQL Console
\password leslie_postgres3  # enter new password 123456
\q  #leave the PostgreSQL Console
```

### 创建数据库
```
sudo -u postgres createdb -O leslie_postgres3 testdb2
or
su - postgres createdb -O leslie_postgres3 testdb2
```

<!--more-->
## 登录数据库
```
psql -U dbuser -d exampledb -h 127.0.0.1 -p 5432
# -U username
# -d database name
# -h ip
# -p port
```

## 常见console命令
```
\h：查看SQL命令的解释，比如\h select。
\?：查看psql命令列表。  # 按q返回
\l：列出所有数据库。
\c [database_name]：连接其他数据库。
\d：列出当前数据库的所有表格。
\d [table_name]：列出某一张表格的结构。
\du：列出所有用户。
\e：打开文本编辑器。
\conninfo：列出当前数据库和连接的信息。
\q: 离开console
\i: 从指定的文件中读取命令 比如:\i basics.sql
直接使用SQL命令操作数据库
```
## 新特性
* 窗口函数
```
SELECT depname, empno, salary, **avg(salary) OVER (PARTITION BY depname)** FROM empsalary;
```
功能和group by 有点类似
* 继承
```
CREATE TABLE capitals (
  state      char(2)
) INHERITS (cities);
```
这样capitals表继承了cities表
在cities表查询数据的时候会获得所有的数据，如果用 only cities则只会包含cities表中的数据

## 开发接口
标准的ODBC和JDBC
python 开发接口：psycopg模块
官方[教程](http://initd.org/psycopg/)
入门[教程](http://www.yiibai.com/html/postgresql/2013/080998.html)
