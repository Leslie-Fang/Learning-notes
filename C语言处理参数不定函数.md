---
title: C语言处理参数不定函数
date: 2018-08-09 20:46:08
tags: 技术
---
使用va函数
示例：求N个数的和

int sum(int count, ...)
{
    int sum = 0;
    int i;
    va_list ap;
    va_start(ap, count);
    for (i = 0; i < count; ++i)
    {
        sum += va_arg(ap, int);
    }
    va_end(ap);
    return sum;
}

下面是 <stdarg.h> 里面重要的几个宏定义如下：
typedef char* va_list;
void va_start ( va_list ap, prev_param );
type va_arg ( va_list ap, type );
void va_end ( va_list ap );
va_list 是一个字符指针，可以理解为指向当前参数的一个指针，取参必须通过这个指针进行。
<Step 1> 在调用参数表之前，定义一个 va_list 类型的变量，(假设va_list 类型变量被定义为ap)；
<Step 2> 然后应该对ap 进行初始化，让它指向可变参数表里面的第一个参数，这是通过 va_start 来实现的，第一个参数是 ap 本身，第二个参数是在变参表前面紧挨着的一个变量,即“...”之前的那个参数；
<Step 3> 然后是获取参数，调用va_arg，它的第一个参数是ap，第二个参数是要获取的参数的指定类型，然后返回这个指定类型的值，并且把 ap 的位置指向变参表的下一个变量位置；
<Step 4> 获取所有的参数之后，我们有必要将这个 ap 指针关掉，以免发生危险，方法是调用 va_end，他是输入的参数 ap 置为 NULL，应该养成获取完参数表之后关闭指针的习惯。说白了，就是让我们的程序具有健壮性。通常va_start和va_end是成对出现。
http://www.cnblogs.com/hanyonglu/archive/2011/05/07/2039916.html
https://blog.csdn.net/xyang81/article/details/41223527
