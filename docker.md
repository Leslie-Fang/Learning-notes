---
title: docker
date: 2017-09-21 19:20:39
tags: 技术
---
## 基本概念
* docker daemon
* image
* container
* network
* label:https://docs.docker.com/engine/userguide/labels-custom-metadata/
* storage driver:http://dockone.io/article/1765
* volumes

## 安装
centos上面用这个区安装
https://docs.docker.com/install/linux/docker-ce/centos/

像Suse这样的OS找不到镜像的话，可以直接下载二进制包去安装
https://docs.docker.com/install/linux/docker-ce/binaries/
参考docker-ce 里面的Binaries下面的方法
安装之后，follow这个链接的最下面去创建systemd的配置项
https://docs.docker.com/config/daemon/systemd/#httphttps-proxy


## 常用命令
* docker ps: 查看正在运行的container
* 通过dockerfile去定义image： https://docs.docker.com/get-started/part2/#apppy
* dockerbuild 利用dockerfile去生成image  docker build -t friendlyhello .  (friendlyhello 镜像名)
**important**: 在dockerfile里面用pip去安装包的时候也需要配置代理
* docker images 查看有没有生成镜像
* docker run -p 4000:80 friendlyhello (-p 4000:80 把container的80端口映射到 主机的4000端口)
docker run -d -p 4000:80 friendlyhello（-d 参数表示运行在后台，in detached mode）
* attach a container
docker attach container_name
这样就会进入contaienr内部
要deattach这个container，Ctrl+P+Q
* docker container ls(只显示activate的container)
查看所有的container: docker container ls -a
查看到container id
* 停止正在运行的container
docker stop container_id
在容器内部直接exit
* 删除container
docker rm container_id
* 删除image
docker image rm image_id（必须要保证这个image没有container在使用）
* copy文件
docker copy * #copy文件

### 常用的一系列用法
下载好image以后
docker run -tid imagename
docker container ls
docker attach container-id #进入容器
host上docker cp等进行修改
docker commit container-id new-name #保存成新的镜像
#### Error messgae:
commit的时候看到error message
Error response from daemon: Error processing tar file(exit status 1): unexpected EOF
看dockerd的日志，是zip文件不能保存


docker save -o update1.tar update //镜像的导出
docker load  < update1.tar //镜像的导入
### 删除文件,减小镜像大小
在container中删除文件之后，再commit发现镜像大小不变
是因为docker commit会保留原来的layer的信息

用export
docker export container_id > name_tag.tar
再次导入
docker import name_tag.tar name:tag

### 无法使用numa的问题
1. docker run -itd --privileged image
--privileged可以给container权限
推荐用这个方法

2. 方法二
docker run -itd --cap-add=SYS_NICE image
当方法1的performance掉了的话，用这个方法试试看

### 代理设置
为docker-run设置代理
https://docs.docker.com/config/daemon/systemd/#httphttps-proxy

为docker容器本身设置代理
和普通OS一样直接设置代理，试过可以工作
export http_proxy=http://***
export https_proxy=https://***

在启动container的时候可以添加参数
https://docs.docker.com/network/proxy/

http://www.vadmin-land.com/2018/09/using-docker-behind-a-proxy/

## 设置docker的工作目录
```
vi /etc/docker/daemon.json
```
And add the following to tell docker to put all its files in this folder, e.g:
```
{
  "graph":"/workspace"
}
systemctl restart docker 之后
```
docker的工作目录就到了/workspace的目录下面

### 镜像目录
/workspace/docker_data/images

### container目录
/workspace/docker_data/containers/

### 数据存储
/workspace/docker_data/volumes

* 管理命令
docker volume ls

docker volume rm volume_name
* 指定volume的挂载位置
指定挂在位置
docker run -itd -v /data/:/data1 centos  bash // -v 用来指定挂载目录,  
-v的格式为<host path>:<container path>

## 上传镜像
docker login
位image添加标签  docker tag image username/repository:tag
docker tag test1 lesliefang/get-started:v.0

docker image ls
会看到这个lesliefang/get-started:v.0 image

上传
docker push username/repository:tag
登录docker hub 可以看到相关的信息
<!--more-->
##　创建services:
scale our application by running this container in a service
用service可以实现负载均衡
一个service其实就是定义了一个镜像如何跑（运行在哪个端口，跑多少个container）
用docker-compose.yml文件去定义:

创建完compose文件之后
* docker swarm init（多张网卡的话：docker swarm init --advertise-addr 10.239.182.67）
* docker stack deploy -c docker-compose.yml appname(自己定义一个名字)
* 查看正在运行的service
docker service ls
* 查看每个容器进程的process id
docker service ps service_id
* docker inspect container_name(通过docker container ls可以查看)
* docker inspect --format{} container_name 来限制输出的内容

### scale利用swarm进行伸缩
* 直接修改docker-compose.yml文件中的replicas的个数
然后重启docker： docker stack deploy -c docker-compose.yml appname
不需要停止原来的app，热更新
* 停止stack
docker stack rm appname 删除了service，但是container还在
docker swarm leave --force 删除container

##  Swarms:
多台机器(实体机或者虚拟机)
有一台机器作为swarm manager,所有的命令都在swarm manager上运行
Swarm managers are the only machines in a swarm that can **execute your commands**, or authorize other machines to join the swarm as workers. Workers are just there to provide capacity and do not have the authority to tell any other machine what it can and cannot do.

在swarm manager上运行docker swarm init（多张网卡的话：docker * swarm init --advertise-addr 10.239.182.67）使能swarm模式。在其它节点的机器上面运行docker swarm join使其它机器加入swarm作为workers
docker swarm init 会返回告诉你docker swarm join的命令里面的token怎么敲

在其它机器上运行docker swarm join，让其它机器加入
如果没办法加入，首先确保关闭manager的防火墙：
http://www.jianshu.com/p/3ab849248b52

docker的各个端口的作用：
https://www.digitalocean.com/community/tutorials/how-to-configure-the-linux-firewall-for-docker-swarm-on-centos-7

在公司电脑上是有可能是代理的问题：
https://github.com/moby/moby/issues/34825

在/etc/systemd/system/docker.service.d/
[Service]
Environment="http_proxy=http://child-prc.intel.com:913" "NO_PROXY=localhost,127.0.0.1,10.239.182.67"
把manager的地址添加到no_proxy里面

在manager上运行docker node ls查看机器个数


##  stack:
A stack is a group of interrelated services that share dependencies, and can be orchestrated and scaled together.

在docker-compose.yml文件的services的tag下面再去定义一个service
多个service一起部署


##  Kubernetes
作用和docker swarm类似（解决容器编排问题），但是比swarm更强大
### 基本概念
* pod：容器的集合，一个pod可以包含一个或多个container。一个pod内运行相同业务的容器，一个pod只能运行在一台机器上
* replicateion controller(rc) 管理pod，保证任何时间有特定数量的pod在运行，多了删除pod，少了创建pod
* service：将pod提供的服务暴露到外网
* label： 用于区分 pod,rc,service的key/value pair
* kubectl: 命令行,调用kubernetes的API
* Kubernetes master: 运行在主节点上，收集三个进程的信息：: kube-apiserver, kube-controller-manager and kube-scheduler
* 其它非主节点运行两个进程：kubelet(与主节点通信), kube-proxy(网络代理，反映了各个节点的kubernetes的网络状况)
* Kubernetes Control Plane
k8s里面所有组件都叫做对象，Kubernetes Control Plane在任意时间都会监测这些对象，保证对象的数量和目标状态
* etcd 是 CoreOS 团队发起的一个管理配置信息和服务发现（service discovery）的项目
http://www.infoq.com/cn/articles/coreos-analyse-etcd

### install
我们是在cluster上装：
https://kubernetes.io/docs/getting-started-guides/centos/centos_manual_config/
**这个文档不能用了，有错**
问题1：当前k8s还不支持docker17版本
需要用docker1.12版本，https://stackoverflow.com/questions/44891775/kubernetes-installation-on-centos7

所以需要首先卸载docker17版本，直接用k8s的安装手册，上面会安装合适版本的docker
老版本的docker的命令和新版本不太一样

现在推荐用kubeadm去enable，过程和swarm有点像
https://kubernetes.io/docs/setup/independent/create-cluster-kubeadm/
http://tonybai.com/2016/12/30/install-kubernetes-on-ubuntu-with-kubeadm/


###  比较容器调度方案
docker swarm VS k8s
http://dockone.io/article/1138
http://www.jianshu.com/p/07daa3a16878
