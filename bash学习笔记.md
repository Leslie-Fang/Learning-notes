---
title: bash学习笔记
date: 2017-06-12 18:55:33
tags: 技术
---
## diff
diff 用于比较两个文件（也可以用于目录）的差别
```
diff file_1_1.txt file_1_2.txt
```
常规模式输出：
* a为后一个文件比前一个文件多的行
* d为后一个文件比前一个文件少的行
* c为两个文件不一样的行


```
diff file_1_1.txt file_1_2.txt -y
```
比较模式输出：按每一行对比两个文件


## vim查找字符串
### 输入查找
在命令模式下输入 ？或者/
然后输入要查找的单词
按回车开始查找，按n查找下一个(上一个)单词
N与n的反方向进行搜索

### 根据光标位置的单词查找
在光标位置的单词按*或者#
按n查找下一个(上一个)单词，N与n的反方向进行搜索

<!--more-->
## 后台运行
Mthod1: ./cycle.sh 200 chassis31_6 AC 2>&1|tee chassis31_6.log &
在命令的最后加一个&，表示希望这个命令在后台运行
Method2: 使用screen

## $?
$? 表示上一个进程或者函数运行之后的返回值
在写bash函数的时候，函数里面return返回值
在调用这个函数之后再调用一次 $? 表示读取一次返回值，**注意$?只能用一次**

## 重定向> /dev/null 2>&1
命令 > /dev/null 2>&1
命令 1> /dev/null 2>&1
两条都不会输出任何东西，表示将错误定向到标准输出，又将标准输出定向到null
这里的&表示后面的1是文件描述符（标准输出），并不是文件名

## 运行脚本``
 \`command\`
键盘最左上角的字符，也就是markdown里面的代码的符号
表示运行\`\`之间的command

## grep匹配行首
`grep "^${group}" local.cfg | awk -F: '{print $2}'`
^表示匹配行首

## 匹配正则表达式 =~
[[ $phrase =~ $keyword ]]
正则表达式


## bash数组
申明：declare -a arr
```
1. ${arr[*]}         # All of the items in the array
2. ${!arr[*]}        # All of the indexes in the array
3. ${#arr[*]}        # Number of items in the array
4. ${#arr[0]}        # Length of item zero
```

构造数组
arr=(element1 element2 ... element3)

为数组添加新的元素
arr=(${arr[@]} elementNew)
for example: chassisarr=(${chassisarr[@]} $group)
意思是，取出原有数组中的所有元素加上新元素赋值回数组

## Top命令
http://www.cnblogs.com/peida/archive/2012/12/24/2831353.html

查看进程包含的线程的信息
首先top看到进程的pid
然后:top -p PID -Hd1
