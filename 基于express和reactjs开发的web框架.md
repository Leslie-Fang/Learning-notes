---
title: 基于express和reactjs开发的web框架
date: 2017-07-23 18:23:22
tags: 技术 web
---
前几个月基于express和reactjs写过几个简单的web应用，现在回过头来看很多代码都想不起来了。这周利用下班时间重新将这个框架梳理了一下，并写下这篇文章作为基于这两个框架开发web应用的BKM。
## 使用到的主要工具介绍
* Express：基于nodejs的后端框架
* Reactjs+Redux：前端框架，React更多的是扮演着MVC中的view的角色，Model以及Control类似的功能可以用flux或者更进一步用Redux来实现，[链接](https://github.com/buckyroberts/React-Redux-Boilerplate) 里面的图片将redux和reactjs之间的关系描述的很清楚
* Babel：转换ES6以及JSX的语法到ES5，commonJS
* Webpack: 浏览器不支持commonJS，因此需要通过webpack打包之后，才可以将js嵌入到html当中，当然经过webpack之后在生产环境中最好再用gulp经过一次uglify，可以减小js文件的大小
* Gulp：前端开发自动化的利器，在开发过程中可以监听文件的变化重启服务器
* 数据库：目前使用MySQL存储用户信息，使用Redis存储session的hotdata。下一步可以把mongodb的接口移植过来。

## 框架实现的基础功能
这是个基础框架，不涉及任何的业务逻辑。在这个框架的基础上可以根据实际需求，开发主页面和相关的业务逻辑。
* 注册和登入功能：使用bcrypt模块对密码进行加密，用户名和密码存储在MySQL里面进行用户信息的验证。验证通过、用户登入之后在cookie中设置session id对应的session数据，并将session的用户热点数据储存在Redis里面。
* 登出功能：通过销毁和重新建立session，来实现用户的登出功能。redis的session数据可以再redis里面设置数据自动销毁的超时时间。当然cookie中的session id也可以通过设置cookie的超时销毁时间来实现。
* 用户登入状态控制：用户的每一次访问根据cookie中的session id进行用户登录状态的控制，未登录用户无法看到定制的信息。
<!--more-->
* 业务逻辑：根据业务需求在main页面中进行业务逻辑的开发

## Reactjs + Redux的前端框架介绍
框架的[介绍](https://github.com/buckyroberts/React-Redux-Boilerplate)
这里介绍一下我在写这个框架的时候的一些BKM：
* 一般在javascript目录下面建立三个目录：一个react目录，是开发的前端代码；一个babel目录：是将react目录下代码经过babel转译之后得到的代码;一个webpack目录：是将babel目录下需要嵌入在html中的代码打包之后的代码。

* 其中只在react目录下开发我们的代码：react的根目录下面：有一个store.js文件，这个是在redux中存储应用状态和数据的。除了store.js之外的文件都是component文件，就是html页面的显示元件，component文件可以通过container目录下面的各个container文件包含的模块化的组件拼凑出来，因此我们说reactjs很好的做到了前端组件的模块化和复用。

* react目录下面的container目录包含了就是前端模块化的组件。
react目录下面的action目录下的代码功能类似于MVC模型中的control，就是一些前端的触发函数，在这些函数被执行之后可以dispatch一些action，reducer就在监听这些action，并获得action的返回数据

* react目录下面的reducer目录下的代码，就在监听action目录下代码派发的action，一旦监听到对应的代码，就可以触发页面跳转或者返回状态数据到store当中，这些数据通过store和container绑定的，就会触发container中对应的数据的更新。每一个reducer的作用就是传入上一次的state和action，输出这一次的新的state的对象。export var logout = function(state = {logout:null},action)，这里面state = {logout:null}，只有当第一次调用到的时候才会有{logout:null}，以后每一次调用到都会输入上一次的state对象。

* 一些常见的问题
react：component的名字要大写开头
redux： reducer下面每个case里面返回的对象每一次都需要新建，可以用object.assign(用在单个对象添加属性可以返回新的对象)，用filter也会新建对象(可以用在数组上面)，
redux如何获取服务器端数据库的数据，在action里面写restful的请求给服务器端，需要配置多个action，一个action1发送请求，在action1获得请求的数据处理中dispatch一个数据完成的action

## 使用redux实现撤销undo和再做redo功能
基本的思想，就是在原有的reducer的基础上，新建一个高阶的reducer（什么是高阶：就是指函数返回的值也是函数）
参考这部分代码
```
export var saveCurrentCommentState = function(state = {currentComment:null},action){
    switch (action.type) {
        case 'SAVE_COMMENT':
            console.log('return the saveCurrentComment',state, 'and action', action);
            //console.log(action.payload.username);
            //console.log(Object.assign({},state,{username:action.payload.username}));
            return Object.assign({},state,{currentComment:action.payload.comment});
        default:
            console.log('return the defalut saveCurrentComment',state, 'and action', action);
            return Object.assign({},state);
    }
};
var highOrdersaveCurrentCommentState = function(reducer){
//for pastComment currentComment and futureComment: each one is a state or state array
    const initialState = {
        pastComment: [],
        currentComment: reducer({currentComment:null}, {}),
        futureComment: []
    };
    return function (state = initialState, action) {
        const { pastComment, currentComment, futureComment } = state;

        switch (action.type) {
            case 'UNDO_COMMENT':
                console.log('return the highOrdersaveCurrentCommentState', state, 'and action', action);
                //console.log(action.payload.username);
                //console.log(Object.assign({},state,{username:action.payload.username}));
                const newpastComment = pastComment.slice(0, pastComment.length - 1);
                const previousComment = pastComment[pastComment.length - 1];
                futureComment.push(currentComment);
                console.log("=========>");
                console.log(previousComment);
                return {
                    pastComment: newpastComment,
                    currentComment: previousComment,
                    futureComment: futureComment
                };
                //return Object.assign({}, state, {comment: action.payload.comment});
            case 'REDO_COMMENT':
                console.log('return the highOrdersaveCurrentCommentState', state, 'and action', action);
                pastComment.push(currentComment);
                const nextComment = futureComment[futureComment.length - 1];
                const newfutureComment = futureComment.slice(0, futureComment.length - 1);
                return {
                    pastComment: pastComment,
                    currentComment: nextComment,
                    futureComment: newfutureComment
                };
            default:
                console.log('return the defalut highOrdersaveCurrentCommentState', state, 'and action', action);
                const newcurrentComment = reducer(currentComment, action);
                if (newcurrentComment === currentComment) {
                    return state
                }
                pastComment.push(currentComment);
                return {
                    pastComment: pastComment,
                    currentComment: newcurrentComment,
                    futureComment: []
                }
        }
    }
};
export var undoRedoCommentState = highOrdersaveCurrentCommentState(saveCurrentCommentState);

```
其中saveCurrentCommentState这个reducer是一个基本的reducer，每一次正常的状态更新都会触发这个reducer来更新对应的状态
定义一个高阶函数highOrdersaveCurrentCommentState，这个函数传入的参数是一个reducer(In)，返回的也是一个reducer(Out)。
定义一个新的reducer，undoRedoCommentState，这个reducer就是通过调用 highOrdersaveCurrentCommentState(saveCurrentCommentState)得到的返回的reducer。
redo和undo的功能都是在这个highOrdersaveCurrentCommentState高阶函数里面实现的，我们现在来看看这个函数里面实现了什么。这个函数只在初始化的时候被调用了一次，首先定义reducer(Out)的初始化的initialState ，并传入到返回的reducer中。在返回的这个reducer中：
1. 如果没有监测到undo或者redo的action，就会触发default的操作，在default操作中会先调用saveCurrentCommentState这个基本的reducer看看是否有正常的状态转移（也就是判断一下是不是状态转移的action触发的，还是和当前的reducer没有半毛钱关系的action触发的），如果是正常的状态转移，那么当前的state加入到past里面，更新当前的state。
2. 如果监测到undo的action，就会把当前的state放入到future中，从past里面取一个state出来放到当前的state里面
3. 如果监测到redo的action，就把当前的state放入到past里面，从future里面取状态回来。

主要的实现逻辑就是这些。当然在这部分代码里面还可以进一步添加，undo的时候是不是past里面存有过去的状态（即是不是完全popout的），redo的时候也需要看看future里面的状态是不是完全popout的。

## 代码的链接
github托管的代码的链接[地址](https://github.com/Leslie-Fang/express_reactjs)
其中相对于master分支，develop分支实现了用户数据在session中的储存，用户的登出功能。

## 下一步开发计划
* 用户登入部分改成https的请求
