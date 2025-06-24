---
title: python调用命令行脚本
date: 2017-07-19 19:45:49
tags: 技术
---
阻塞态调用和非阻塞态调用，这两个名字是自己根据调用的特点给区分的。
两者主要的区别在于是否会另外开辟一个子进程去调用这些命令行的脚本。在所谓的阻塞态调用下，python会等待这个脚本执行完毕再顺序往下执行其它的程序。
在所谓的非阻塞态调用下，python则会开辟一个子进程，将脚本放在子进程里面执行，自己则立刻向下运行。
## 阻塞态调用
使用os模块，主要涉及到popen和system两种方法。
这两种方法的区别参考一下[链接](http://blog.csdn.net/windone0109/article/details/8895875)
os.system(cmd) 返回的是程序运行的结果状态，比如程序运行成功，则返回0，有错误则返回其对应的错误代码
os.popen(cmd) 返回的是程序输出的结果，比如cmd=“ls”的时候，通过popen就可以得到ls的结果

```
cmd = "echo hello wolrd"
ret = os.popen(cmd)
information = os.system(cmd)
```

## 非阻塞态调用
非阻塞态调用主要用到了subprocess 这个[模块](https://docs.python.org/2/library/subprocess.html)
简单的用法就用subprocess.call(cmd)就可以了
更高级复杂的用法可以使用subprocess.popen(cmd)
比如说有时候因为环境变量的配置我们需要在特点的目录下执行脚本，就可以使用subprocess.popen的cwd参数,要配置环境变量而不是继承原有的环境变量可以使用env参数
```
workingDirector = r"C:/testcode"
cmd = r"test.bat"
subprocess.popen(cmd,cwd=workingDirector)
```
