---
title: tensorflow_benchmark_机制分析
date: 2020-09-26 21:47:29
tags: 技术 深度学习
---
## 介绍
本文介绍Tensorflow Op 级别的benchmark的用法以及代码调用

## 用法
```
* 编译并运行
bazel --output_user_root=$build_dir run --config=mkl --copt=-O3 //tensorflow/core/kernels/image:non_max_suppression_op_benchmark_test --benchmarks=../

* 先编译再运行
bazel --output_user_root=$build_dir build --config=mkl --copt=-O3 //tensorflow/core/kernels/image:non_max_suppression_op_benchmark_test
设置环境变量
TF_NUM_INTEROP_THREADS=1 TF_NUM_INTRAOP_THREADS=28 OMP_NUM_THREADS=28
numactl --localalloc --physcpubind 0-27 ./bazel-bin/tensorflow/core/kernels/image/non_max_suppression_op_benchmark_test --benchmarks=../

* 编译静态链接可移植的二进制
bazel --output_user_root=$build_dir build --config=monolithic --copt=-O3 //tensorflow/core/kernels/image:non_max_suppression_op_benchmark_test
```

## 代码分析
### Benchmark文件
https://github.com/tensorflow/tensorflow/blob/e8598ce0454c440fca64e4ebc4aeedfa7afd5c97/tensorflow/core/kernels/image/non_max_suppression_op_benchmark_test.cc
```
#define BM_CombinedNonMaxSuppressionDev(DEVICE, B, BN, CN, Q)                \
  static void BM_CombinedNMS_##DEVICE##_##B##_##BN##_##CN##_##Q(int iters) { \
    testing::ItemsProcessed(iters* B);                                       \
    test::Benchmark(#DEVICE, BM_CombinedNonMaxSuppression(B, BN, CN, Q))     \
        .Run(iters);                                                         \
  }                                                                          \
  BENCHMARK(BM_CombinedNMS_##DEVICE##_##B##_##BN##_##CN##_##Q);
```

### 封装构造testing::Benchmark
#### Step1:
BENCHMARK(BM_CombinedNMS_##DEVICE##_##B##_##BN##_##CN##_##Q);

#### Step2 注册benchmark 进入all_benchmark这个全局变量:
https://github.com/tensorflow/tensorflow/blob/e8598ce0454c440fca64e4ebc4aeedfa7afd5c97/tensorflow/core/platform/default/test_benchmark.h#L28-L33

* TF_ATTRIBUTE_UNUSED __attribute__((unused)) 忽略GCC 告警： https://www.jianshu.com/p/21aef14340a8
* new ::tensorflow::testing::Benchmark(#n, (n)) 将这个函数注册到all_benchmark这个全局变量里面 里面 https://github.com/tensorflow/tensorflow/blob/e8598ce0454c440fca64e4ebc4aeedfa7afd5c97/tensorflow/core/platform/default/test_benchmark.cc#L39-L43

<!--more-->

#### Step3 注册之后进入test_main执行benchmark
这一部分我理解应该是在bazel build的时候加到non_max_suppression_op_benchmark_test.cc 里面的
* test_main.cc:main
* test_benchmark.cc:Benchmark::Run(const char* pattern)
遍历所有all_benchmark 一个一个去执行，并打印出每个的运行结果
```
b->Run(arg.first, arg.second, &iters, &seconds); #这个b就是具体的某个benchmark
```
* test_benchmark.cc:Benchmark::Run(int arg1, int arg2, int* run_count, double* run_seconds)  
```
(\*fn0_)(iters);
```
这里就调用回到了执行这个函数
```
BM_CombinedNMS_##DEVICE##_##B##_##BN##_##CN##_##Q(int iters)

test::Benchmark(#DEVICE, BM_CombinedNonMaxSuppression(B, BN, CN, Q))     \
        .Run(iters); #进入下面的分析实际运行test::benchmark
```

### 实际运行test::benchmark

```
test::Benchmark(#DEVICE, BM_CombinedNonMaxSuppression(B, BN, CN, Q))     \
        .Run(iters);
```

调用进入 https://github.com/tensorflow/tensorflow/blob/e8598ce0454c440fca64e4ebc4aeedfa7afd5c97/tensorflow/core/common_runtime/kernel_benchmark_testlib.cc#L136
