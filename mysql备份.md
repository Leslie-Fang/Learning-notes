---
title: mysql备份
date: 2019-01-18 19:03:16
tags: 技术
---
# 冷备份
## 生成sql文件
mysqldump -u用户名 -p 数据库名 > 数据库名.sql
范例：
mysqldump -umanzuo -p -h116.62.126.101 manzuoTestENV > manzuoTestENV.sql

## 导入sql文件
mysql -u用户名 -p 数据库名 < 数据库名.sql
范例：
mysql -uroot -p manzuoTestENV < manzuoTestENV.sql
