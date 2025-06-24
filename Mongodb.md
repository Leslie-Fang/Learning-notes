---
title: Mongodb
date: 2017-04-16 11:35:46
tags: 技术
---
## 基本术语
collection 对应了 table
document 对应了 row
field 对应了 column
database，index，primary key都是一致的

## Macos
用homebrew安装，

运行 mongodb：两种方式
方式1：打开mongodb的图形化界面，默认连接的数据目录的配置文件在/usr/local/etc/mongod.conf
可以修改/data/db的路径

方式2： 命令行下，在shell中进入安装目录，我的mac：
/Applications/MongoDB.app/Contents/Resources/Vendor/mongodb
将这个目录配置到PATH环境变量里面
配置数据目录 默认路径为/data/db,运行 sudo mongod(需要sudo，因为/data/db在根目录),这样就打开了mongodb，可以等待连接了
或者在用户目录~下建立~/data/db,然后在启动mongod时：mongod -dapath ~/data/path
<br />

管理mongodb：两种方式
1. 命令行运行mongo
把上面的路径配置到PATH下面，运行mongo，进入mongo-shell管理界面
可以用show dbs显示所有数据库
use db_name使用某个数据库
```
# 查看collection中的所有记录
db.collection.find()
# 查看collection中的记录的数量
db.collection.count()
```

2. 用用MongoChef图形化界面进行管理
运行mongo进入管理界面之后，

<!-- more -->
## Nodejs接口
Nodejs下面有两种模块的接口支持对MongoDB的访问，mongodb的[官网使用的是mongodb](https://docs.mongodb.com/getting-started/node/client/), 但是在实际生产环境使用Mongoose更多一点

### mongodb
mongodb的使用和在mongo的命令行从操作数据库比较类似
用GraphQL和mongondb写过一些操作的例子，具体的语法可以参考[官网的教程](https://docs.mongodb.com/getting-started/node/client/)，或者[github代码链接](https://github.com/Leslie-Fang/express_GraphQL)

### Mongoose
[官网](http://mongoosejs.com/)
[中文教程](http://www.cnblogs.com/zhongweiv/p/mongoose.html)
在nodejs下面可以用Mongoose模块进行mongodb管理
schema，model，的概念，先定义一个schema对应了数据表(collection)的结构，然后用schema来创建一个一个model，使用model来具体操作数据(document),代码的组织可以参考这个[中文教程](http://www.cnblogs.com/zhongweiv/p/mongoose.html)
一般会在一个独立的文件中新建一个schema以及model，然后将这个model export出来
在需要用到的地方require这个model
1. 保存数据
  document=new model（data）
  传入数据，创建一个新的document（一行数据）
  document.save（function（）{

    }）
  调用save保存到数据库中，在回调函数中进行处理

2. 更新数据,三个参数
  model.update(query，data，callback)
  1.query是匹配查找你想要更新哪一行数据(哪个document)
  2.data是你希望更新的数据1.set修改数据2.如果数据的数组，可以往里面push，pop一个element
  3.callback，是更新结束之后的回调函数

3. 查询
  model.find(function(){})
  在回调函数中进行查询到的数据的处理

## Pymongo
在python下面可以通过pymongo模块进行mongodb的管理
```
### 连接数据库
client = pymongo.MongoClient(dbConfig['url'], dbConfig['port'])
### 获取数据库
db = client['quant'] # same as 'db = client.quant'
### 获取Collection
collection = db['tradeHistoryData']
### 创建数据库
db = client['quant'] #如果数据库不存在会自己创建
### 创建collection
collection = db['tradeHistoryData'] # collection 不存在则会被创建
### 插入数据
#### 单挑插入
document1 = {} ### 创建数据，字典类型
post_1 = collection.insert_one(document1).inserted_id ##其中.inserted_id将返回ObjectId对象
#### 批量插入
new_posts = [{document1},{document2}]
result = collection.insert_many(new_posts)

### 获取单条数据(document)
collection.find_one()

### 获取所有document
lines = collection.find()
lines = collection.find({"author": "Mike"}) ## 添加查找的条件
collection.find({"author": "Mike"}).count ## 计数

### 创建索引
result = collection.create_index([('user_id', pymongo.ASCENDING)], unique=True)

### 更新数据
collection.update_one({'x':4},{'$set':{'x':3}}) ##其中传入的第一个参数是你想要更新的数据，第二个是你想要更新的最新数据。其中$set部分是必要元素，如果没有会报出错误。除了$set外还有很多其它的比如$inc，对应着不同的功能

### 删除数据
collection.delete_one({'x':3})

### 关闭连接
client.close()
```
## 数据库冷备份和还原
参考这篇[文章](http://blog.51yip.com/nosql/1573.html)
可以冷备份还原数据库；数据库中的指定表；指定表的指定字段
