---
title: Django
date: 2017-04-15 14:01:54
tags: 技术
---
# 开发过程
新建一个app之后，在settings.py的INSTALLED_APPS中需要注册一下这个APP
## 模板的配置
在app的目录下面新建一个tempaltes的目录
在base_dir/app_name/templates/app_name/xxx.html这个目录下面去写html
然后在setting.py的TEMPLATES的'DIRS'加上，[os.path.join(BASE_DIR, app_name, 'templates'),
这样在app下面的views里面就可以render这个html了，然后在url里面做配置

## 静态文件的配置
在app的目录下面新建一个static的目录
在base_dir/app_name/static/app_name/目录下面去建imgaes，css,js等目录
在下面摆放静态文件
在settings.py按照模板配置

## 集成ajax
主要是要注意：在用ajax提交表单的时候，需要在header中包含进csrf-token

## 集成reactjs
Django默认使用的templates比较适合用在静态页面(页面不太变化的场景下面)，所有的html中的数据都通过context传进去。
Reactjs则比较适合用在动态页面中，一般会在项目的根目录下建立一个目录(front)专门用于储存reactjs的代码，同时将reactjs的code经过webpack之后的代码放入另一个目录中(webpack)。在Django的setting中static文件需要包含从这个webpack目录中去找。
在Django的html文件中：1.需要包含reactjs渲染的主块 2.另外reactjs的代码express中是通过script包含进去的，在django中则是使用django webpack load去包含进去。

同时在webpack中可以配置生成的文件用hash之后的名字命名。这样在生产环境中可以避免浏览器缓存的问题，就是浏览器缓存了上一版本的js文件，用hash之后的文件名就会重新加载。webpack会生成一个json文件，里面包含了webpack前后的文件名的对应关系，这样在html中引入webpack之后的hash名的js文件的时候，就不需要没有刺激都修改引入的文件名了。
比如：通过webpack-bundle-tracker会写入webpack-stats.json文件中
1. python manage.py runserver 来启动django
2. 敲gulp 来生成babel和webpack的文件
3. webpack配置webpack-bundle-tracker来生成webpack-stats.json文件


## 部署
部署到阿里云上，创建好数据库之后，启动，会报错说数据表不存在，python manager.py makemigrations 也会报同样的错
有两种可能的解决方法：
1. 数据迁移
把开发环境中的数据都迁移到云端，mysql比较方便，用冷迁移就可以了
备份数据: mysqldump -u root -p databaseName > data.sql
恢复: mysqldump -u root -p  databaseName < data.sql

当然mysql也支持数据的热备份，只需要在mysql中配置就可以了，本质上就是在A服务器上对数据库操作的sql语句会备份在一个log文件中，B服务器会监听这个log文件，实时更新到本地，并执行对应的sql语句

2. 将各个APP下面的migration文件夹中的00x开头的记录django数据model变化的文件删除，然后重新migrate一下
