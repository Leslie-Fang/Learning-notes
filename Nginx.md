---
title: Nginx
date: 2017-09-25 19:10:55
tags: 技术
---
## 介绍
一些web框架入django,flask，express等内部已经集成了一些web服务器，如：web服务器与应用程序交互规范：最早出现的是 CGI，后来又出现了改进 CGI 性能的FasgCGI，Java 专用的 Servlet 规范，Python 专用的 WSGI 规范等等
但是这些web服务器社和用在开发环境不适合用在生产环境。
在生产环境的用法是nginx+uWSGI+web应用框架程序
* 其中web应用程序的框架只提供服务
* uWSGI作为web程序和nginx服务器之间的桥梁
* ngnix作为服务器


## 安装配置
1. 下载安装nginx
官网
https://www.nginx.com/resources/wiki/start/topics/tutorials/install/
添加repo仓库，然后安装，根据平台不同，repo仓库的配置文件有一些对应的字段需要修改
http://nginx.org/en/linux_packages.html
基本的安装以及入门教程
http://www.itzgeek.com/how-tos/linux/centos-how-tos/install-nginx-on-centos-7-rhel-7.html

ubuntu:http://wiki.ubuntu.org.cn/Nginx
通过apt-get install 安装
也可以通过systemctl start(stop,reload,restart) nginx来控制ngnix
配置文件目录：/etc/nginx
log文件目录：/var/log/nginx
自带的html目录: /usr/share/nginx/html

2. uWSGI
uWSGI的介绍：https://github.com/unbit/uwsgi
  * 安装
  pip install uwsgi
  报错: no python.h file，https://github.com/MeetMe/newrelic-plugin-agent/issues/151
  先装python-devel和
  装 python-devel,https://stackoverflow.com/questions/23215535/how-to-install-python27-devel-on-centos-6-5
  先找到适合的包，yum search python | grep -i devel
  然后Yum install

  * 配置
  快速入门教程: http://uwsgi-docs.readthedocs.io/en/latest/WSGIquickstart.html
  这个快速入门教程里面介绍了如何使用:1.uWSGI+web程序框架 2.ngnix+uWSGI+web程序框架（flask,djano等）

3. ngnix
最好的nginx的教程：
http://openresty.org/download/agentzh-nginx-tutorials-zhcn.html

4. 三者集成的一些文章
生产环境不使用flask自带的WSGI服务器，独立配置
http://knarfeh.com/2016/06/11/%E5%86%99%E7%BB%99%E6%96%B0%E6%89%8B%E7%9C%8B%E7%9A%84Flask+uwsgi+Nginx+Ubuntu%E9%83%A8%E7%BD%B2%E6%95%99%E7%A8%8B/
flask官方：http://flask.pocoo.org/docs/0.12/deploying/uwsgi/
入门配置nginx: https://segmentfault.com/a/1190000002411626

<!--more-->
## 实例代码
https://github.com/Leslie-Fang/D3Demo
* 仅仅使用uwsgi:
  uwsgi --http :5000 --manage-script-name --mount /=main:app
  uwsgi各个参数的[含义](http://uwsgi-docs.readthedocs.io/en/latest/Options.html)
  [参考的配置](http://flask.pocoo.org/docs/0.12/deploying/uwsgi/)
  --http 指定端口
  --mount 绑定上特定的APP，其中main.py是文件名字，app是文件名中的APP的名字(app = Flask(__name__))
  --manage-script-name 经常和--mount一起使用

* 使用uwsgi+ngnix
  uwsgi:
  uwsgi --socket 127.0.0.1:5000 --wsgi-file main.py --callable app --processes 4 --threads 2 --stats 127.0.0.1:9191

  --socket: 指定绑定的socket，在ngnix的配置中会使用到
  --wsgi-file： 指定web应用的入口文件
  --callable： 指定入口文件中的APP名字(app = Flask(__name__))
  --stats: 将uWSGI状态作为一个JSON对象导出到一个socket(http://uwsgi-docs-zh.readthedocs.io/zh_CN/latest/StatsServer.html),通过这个socket可以读取uwsgi的信息

  ngnix:
  ubuntu的话/etc/ngnix/ngnix.conf文件
  在http块中添加server块：
  ```
  server{
          listen 8070;
          server_name 0.0.0.0;
          location / {
                  include uwsgi_params;
                  uwsgi_pass 127.0.0.1:5000;
          }
  }
  ```
## 解决实现负载均衡过程中的session问题
1. nginx使用iphash策略去分配访问流量
iphash策略会将同一个ip的访问分配给同一台服务器，那么同一个用户每一次访问的session就是相同的
缺点：要是再同一个用户访问期间，服务器挂了，这时候nginx会将访问分配给不同的服务器，session不一样了，用户需要重新登录

2. 将session存在cookie里面
将session存在cookie里面，但是要对[cookie数据加密](http://www.jianshu.com/p/576dbf44b2ae)，不然session数据不安全
优势：1.不存在session在服务器存储和同步问题 2.比如说这时候有两个web服务器，一个django写的，一个express写的，用户只要在一台服务器上登录成功，带着cookie访问另外一台服务器就自己会登录了
缺点：每一次访问带的cookie数据量变大了，但是其实时间消耗也没多多少

3. 将session存在redis里面，同时不同服务器之间的redis数据热同步
redis之间共享数据也就2ms
缺点：访问峰值的时候可能会很慢

第1，3个方法可以结合使用，第二个方法比较独立(leetcode就是这么实现负载均衡的)


## nginx 配置
[基础配置文档](http://www.nginx.cn/591.html)
```
server{
  server{
          listen 3389;
          server_name 0.0.0.0;
          error_page 404 /40x.html;
          location / {
                  # root /usr/share/nginx/html;
                  index testnginx.html;
          }
          location /index {
                  root /usr/share/nginx/html;
                  index testnginx2.html;
          }
  }
}
```
* root如果不配置，就会默认使用nginx安装过程中的html目录，也就是自带的html目录: /usr/share/nginx/html,这个例子中将root配置成上述的绝对路径也是可以的
* index配置：表示location后面没有带具体的资源的名字的时候(比如这个例子中第一个location的/，就符合条件，没有带上具体访问哪个资源)，则返回这个index配置的资源
* server_name:如果nginx中只配置一个server域的话，则nginx是不会去进行server_name的匹配的。
* 第二个location，如果请求http://47.91.245.251:3389/index/，则会到/usr/share/nginx/html/index目录下面去查找，因为这个location也没有带上具体的资源，所以返回index对应的资源（这个请求和http://47.91.245.251:3389/index/testnginx2.html请求效果是一样的）
