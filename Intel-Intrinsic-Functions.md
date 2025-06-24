---
title: Intel_Intrinsic_Functions
date: 2019-05-12 21:23:38
tags: 技术
---
## 介绍
### SSE
128位的SIMD指令集
https://blog.csdn.net/woxiaohahaa/article/details/51014425
SSE有8个128位寄存器，XMM0 ~XMM7


### AVX2
AVX2 expands most integer commands to 256 bits and introduces fused multiply-accumulate (FMA) operations.
AVX uses sixteen YMM registers. Each YMM register contains:
* eight 32-bit single-precision floating point numbers or
* four 64-bit double-precision floating point numbers.

### AVX512
AVX-512 are 512-bit extensions to the 256-bit Advanced Vector Extensions SIMD instructions for x86 instruction set architecture (ISA)

### 一般的流程
参考这个知乎的回答
https://www.zhihu.com/question/51206237
1. load：将数据从内存载入到寄存器里面
2. 计算
3. save, 将数据从寄存器写入到内存变量里面

### 头文件
Intrinsic functions 是直接提供给客户去调用的
```
#include <immintrin.h>
```
每个Intrinsic函数封装了调用了一些汇编

### api手册
查看这个link
https://software.intel.com/sites/landingpage/IntrinsicsGuide/
打开一个函数，可以看到里面的汇编

### 反汇编查看
示例代码：
https://github.com/Leslie-Fang/Intel_Intrinsic_Functions
在这个示例代码里面，调用了sse的_mm_add_ps(Intrinsic functions)去做加法，objdump -d main反汇编之后可以看到调用了addps的汇编指令，查看上面的API手册_mm_add_ps(Intrinsic functions)函数就是封装了这条指令
