---
title: numpy
date: 2017-09-24 11:12:34
tags: 技术
---
## 使用ipython进行数据分析
ipython有shell的接口也有web的接口
官网：https://ipython.org/
ipython功能:https://www.zhihu.com/question/51467397

运行：jupyter notebook --ip=0.0.0.0 --allow-root

### 特殊用法
* Tab键自动补全
* 内省，变量后跟？，显示变量的信息

## 入门教程：
https://zhuanlan.zhihu.com/p/24988491
书籍：
https://detail.tmall.com/item.htm?_u=om6p4lp35f6&id=36838488413
1. 生成随机数
numpy各种生成随机数的函数:http://www.jianshu.com/p/214798dd8f93
numpy.random.seed
https://www.reddit.com/r/learnpython/comments/3nidns/how_to_use_numpyrandomseed/
用法：numpy.random.seed(num1)
生成随机数
下一次再生成随机数的时候再调用一次numpy.random.seed(num1)
依然生成同样的随机数
如果numpy.random.seed(num2)
再生成随机数就是不同的随机数


2. Numpy.genfromtxt
从文件读取数据
http://www.jianshu.com/p/82110f1dbb94

3. array.tolist
作用:convert a NumPy array to a Python List
https://docs.scipy.org/doc/numpy/reference/generated/numpy.ndarray.tolist.html
b = a.tolist()
重新创建
a = np.array(b)

4. 广播法则
处理不同维度的矩阵之间的运算
https://ptorch.com/news/38.html

5. 从文件读取数据
np.genfromtxt(filename)
http://www.jianshu.com/p/82110f1dbb94
