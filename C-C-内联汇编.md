---
title: C/C++内联汇编
date: 2018-04-08 19:02:45
tags: 技术
---
## 工具
gcc asm去编译汇编，这个叫做汇编器
编译器将C代码编译成汇编代码，汇编器将汇编代码编译成机器码
教程： https://www.ibm.com/developerworks/cn/aix/library/au-inline_assembly/index.html
https://www.jianshu.com/p/1782e14a0766

## 代码AT&T格式汇编
```
#include "stdio.h"
void main(){
  int a=10, b;
  printf("%d",a);
  asm("movl %1, %%eax \n\t"
          "movl %%eax, %0"
          :"=r"(b)           /* output */
          :"r"(a)              /* input */
          :"%eax"         /* clobbered register */
  );
  printf("%d",b);
}
```
编译运行
```
gcc test.c -o test
./test
```

## intel格式汇编
<!--more-->
https://blog.werner.wiki/72829768/
```
#include "stdio.h"
void main(){
  //printf("hello world!");
  int a=10, b=0;
  printf("%d \n",b);
  asm("mov eax,5 \n\t"
    "mov eax,%1 \n\t"
    "mov %0,eax"
    :"=r"(b)
    :"r"(a)
    :"eax"
  );
  printf("%d \n",b);
}
```
编译运行
```
gcc -masm=intel test.c -o test
./test
```
## intel格式汇编与AT&T格式汇编的区别
https://blog.csdn.net/tigerjibo/article/details/7708811
https://blog.csdn.net/tianshuai1111/article/details/7900084

## C转汇编
1,把*.c程序转变为AT&T格式汇编*.s
[root@xxx asm_study]# gcc -S asm.c -o asm.s
[root@xxx asm_study]# ls -al asm.s
-rw-r--r-- 1 root root 1387 06-30 10:41 asm.s

2,把*.c程序转变为Intel格式汇编*.s
[root@xxx asm_study]# gcc -masm=intel test.c -o test.s
当然，要想把c程序转为Intel汇编时，其中不能包含AT&T格式的汇编，否则无法转。

## C++
### AT&T风格
```
#include <iostream>
using namespace std;

void main(){
	int a=10,b=0;
	cout<<b<<endl;
	asm("movl %1, %%eax \n\t"
          "movl %%eax, %0"
          :"=r"(b)           /* output */
          :"r"(a)              /* input */
          :"%eax"         /* clobbered register */
     );
	cout<<b<<endl;
	return;
}
```
编译
```
icpc test.cpp -o test3
./test3
```

### intel风格
```
#include <iostream>
using namespace std;

void main(){
	int a=10,b=0;
	cout<<b<<endl;
	asm("mov eax, %1 \n\t"
          "mov %0, eax"
          :"=r"(b)           /* output */
          :"r"(a)              /* input */
          :"eax"         /* clobbered register */
     );
	cout<<b<<endl;
	return;
}
```
编译运行
```
icpc -masm=intel test.cpp -o test3
./test3
```

## 读取cpuid
原理：
读cpu的信息只需要一条汇编指令 cupid ,入口参数在EAX寄存器，返回的信息在EAX，EBX，ECX，EDX寄存器，也就是说执行cupid之前先给EAX寄存器赋值，在执行cupid，执行过之后，cpu的相关信息就在EAX，EBX，ECX，EDX寄存器中了，入口参数EAX
https://blog.csdn.net/fisher_jiang/article/details/4348194
https://blog.csdn.net/razor87/article/details/8711712
```
#include <iostream>
#include <string>
using namespace std;

void main(){
	//string message;
	int a=10;
	int iEAXValue,iEBXValue,iECXValue,iEDXValue;
	asm volatile("mov eax,0 \n\t"
		"cpuid \n\t"
		"mov %0,ebx \n\t"
		"mov %1,ecx \n\t"
		"mov %2,edx"
		:"=r"(iEBXValue),"=r"(iECXValue),"=r"(iEDXValue)
		:"r"(a)
		:"eax"
	);
	cout<<hex<<iEBXValue<<endl<<iECXValue<<endl<<iEDXValue<<endl;
	return;
}
```
