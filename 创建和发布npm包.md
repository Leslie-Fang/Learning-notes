---
title: 创建和发布npm包
date: 2017-08-05 21:13:16
tags: 技术 Node.js
---
## 创建包
```
//在初始化话时我们定义了module的入口文件是index.js
//在初始化的时候输入的package的名字就是后面npm install时的名字，后面也可以再package.json里面修改
npm init
```

在index.js中定义需要export出来的函数

## 发布包
新用户,先去npm的网站[注册](!https://npmjs.org/)

```
//输入注册npm时的账号密码以及邮箱
npm adduser

//查看信息
npm config ls

//发布
npm publish
```

## 更新包
修改的原有的包的代码之后不可以直接publish的已有的仓库和版本当中,会提示我们没有权限复写已经publish的包
这时候需要修改package.json里面包的版本号

在已经依赖于这个的包的用户这边
```
npm update
```
可以将使用的包更新的最新

## 包的使用
需要使用这个package的时候
```
npm install pkg-name --save(--save-dev)
```

## 删除已经发布的包
```
npm unpublish [<@scope>/]<pkg>[@<version>]
```

## 在Git上维护自己的包
新建一个主分支和一个开发分支
每一部分开发完成开一个版本分支，将版本的分支publish到npm上面

在develop分支上进行开发，开发完成切换到对应的版本的备选分支，review备选分支的代码，没有问题的话merge的备选分支里面
修改package.json里面的版本号，上传到git以及发布到npm
切回到develop分支进行新的开发
