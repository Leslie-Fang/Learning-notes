---
title: CherryPy
date: 2017-05-12 21:59:48
tags: 技术
---
## 介绍
[CherryPy](http://cherrypy.org/)是一个python的web框架
是目前接触过的三种python编写的web框架里(Django,Tornado,CherryPy)最轻量化的
因为公司项目的开发需要，最近开始用每天的业余时间学习一下这个框架设计模式
首先谈一下读完教程配合着写一些简单的应用之后的整体感受，相比于另外两个框架，CherryPy配合着SQLite使用的确做到了轻量化，但是这也意味着框架固定，灵活性不足。
## 安装
```
pip install cherrypy
#使用
import cherrypy
```

<!--more-->
## 框架
搭建最小应用时可以参考我的[仓库](https://github.com/Leslie-Fang/myCherryPy)
### 配置文件
定义一个字典通常命名为conf，可以定义多个不同的配置字典conf1，conf2

### 启动服务
有两种方式：
1.
```
cherrypy.quickstart(webapp, '/', conf)
```

webapp是类名，在这个类中定义RESTAPI
‘/’是对应的URL
conf是对应的配置文件
2.
```
cherrypy.tree.mount(webapp, '/', conf)
#cherrypy.tree.mount(Forum(), '/forum', forum_conf)
cherrypy.engine.start()
cherrypy.engine.block()
```

### 渲染文件
在根目录定义index.html文件
然后定义入口

```
class StringGenerator(object):
    @cherrypy.expose
    def index(self):
        return open('index.html')
```

### RESTFUL API
```
@cherrypy.expose
class myFirstService(object):

    @cherrypy.tools.accept(media='text/plain')
    def GET(self):
        with sqlite3.connect(DB_STRING) as c:
            cherrypy.session['ts'] = time.time()
            r = c.execute("SELECT * FROM STUDENT")
            print r.fetchone()
            return 'hhui'
    def POST(self):
```

### 静态文件
在配置中添加
```
'/static': {
    'tools.staticdir.on': True,
    'tools.staticdir.dir': './public'
}
```
所以根目录下的public文件夹里的东西对应了就是URL-static

### 数据库
CherryPy配合SQLite使用可以搭建轻量化的web应用
通常在Linux发行版本中都会预装SQLite的数据库
定义DB文件的名字
DB_STRING = "testDB.db"
连接数据库||执行CURD操作
```
    with sqlite3.connect(DB_STRING) as c:
    cherrypy.session['ts'] = time.time()
    r = c.execute("SELECT * FROM STUDENT")
    print r.fetchone()
    return 'hhui'
```
