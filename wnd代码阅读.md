---
title: wnd代码阅读
date: 2019-03-16 21:56:54
tags: 深度学习 推荐系统
---
## 两个工具
```
//tensorflow/tools/graph_transforms:transform_graph
//tensorflow/tools/graph_transforms:summarize_graph

看对应的代码
summarize_graph_main.cc
transform_graph_main.cc
```

summarize这个工具代码比较简单，主要用来打印出所有的节点的名字
transform才是用来对计算图进行转换，这个很重要

## 看transform_graph_main.cc
### ParseFlagsAndTransformGraph 解析命令行参数

调用ParseTransformParameters解析--transforms传进来的所有要执行的转换操作
TransformGraph正式执行转换

### ParseTransformParameters
调用scanner去解析--transforms传进来的文本

### TransformGraph
遍历--transforms传进来的所有操作
调用
Status transform_result =
    transform_func(\*graph_def, context, &transformed_graph_def);去执行转换
transform_func时通过GetTransformRegistry仓库拿到的
### GetTransformRegistry
有个TransformRegistry对象
TransformRegistrar类是个仓库
调用REGISTER_GRAPH_TRANSFORM往仓库里面注册transform的操作

代码里grep -rni "REGISTER_GRAPH_TRANSFORM"
可以看到所有注册操作的地方

看具体注册的操作
grep -rni "REGISTER_GRAPH_TRANSFORM(\"fuse_quantized_matmul_and_requantize"


## 一个转换的例子：
```
$ $TF_SRC_ROOT/bazel-bin/tensorflow/tools/graph_transforms/transform_graph \
--in_graph=a.pb \
--out_graph=b.pb \
--inputs='Placeholder*' \
--outputs='Output'  \
--transforms='strip_unused_nodes'
```

### 先看strip_unused_nodes操作
```
-bash-4.2# grep -rni "REGISTER_GRAPH_TRANSFORM(\"strip"
tensorflow/tools/graph_transforms/strip_unused_nodes.cc:192:REGISTER_GRAPH_TRANSFORM("strip_unused_nodes", StripUnusedNodes);
```

## 看模型
运行读取pb
tensorboard --logdir log/
对比INT8和FP32两个模型，发现主要区别在DNN模型上面

## INT8 DNN 模型
TF 代码使用分支:
git checkout remotes/origin/dungeon_wnd

INT8 DNN从tensorboard上面看到调用op是
QuantizedMatMulWithBiasAndReluAndRequantize

在代码里面grep这个op
```
tensorflow/core/kernels/mkl_qmatmul_op.cc:1182:REGISTER_KERNEL_BUILDER(Name("QuantizedMatMulWithBiasAndReluAndRequantize")
tensorflow/core/kernels/mkl_qmatmul_op.cc:1190:REGISTER_KERNEL_BUILDER(Name("QuantizedMatMulWithBiasAndReluAndRequantize")
```
但是这两个op对应的实现都是NoOp
在计算启动的时候进过mkl层添加_Mkl前缀

## 如何具体看MKLDNN的调用关系
### step1
首先export MKLDNN_VERBOSE=1
运行代码，得到mkldnn调用到的具体的kernel的名字，比如igemm_s8u8s32:blas
### step2
在代码里面搜索igemm_s8u8s32:blas，得到对应的执行的代码在gemm_x8s8s32x_inner_product.cpp第62行的generate函数去执行
### step3
编译debug版本的tensorflow，gdb调试，断点打在gemm_x8s8s32x_inner_product.cpp第62行
bt 去看调用栈
```
#0  mkldnn::impl::cpu::gemm_x8s8s32x_inner_product_fwd_t<(mkldnn_data_type_t)6, (mkldnn_data_type_t)6>::pp_kernel_t::generate (this=this@entry=0x32ea900) at external/mkl_dnn/src/cpu/gemm_x8s8s32x_inner_product.cpp:62
#1  0x00007fc231ee0048 in mkldnn::impl::cpu::gemm_x8s8s32x_inner_product_fwd_t<(mkldnn_data_type_t)6, (mkldnn_data_type_t)6>::pp_kernel_t::pp_kernel_t (this=0x32ea900, pd=0x59cd000, dst_is_acc=<optimized out>)
    at external/mkl_dnn/src/cpu/gemm_x8s8s32x_inner_product.cpp:58
#2  0x00007fc231be7613 in gemm_x8s8s32x_inner_product_fwd_t (outputs=..., inputs=..., apd=0x59cd000, this=0x395e300)
    at external/mkl_dnn/src/cpu/gemm_x8s8s32x_inner_product.hpp:118
#3  mkldnn::impl::cpu::gemm_x8s8s32x_inner_product_fwd_t<(mkldnn_data_type_t)6, (mkldnn_data_type_t)6>::pd_t::create_primitive (this=0x59cd000, primitive=0x7fc20f8b4550, inputs=0x7fc20f8b4560, outputs=<optimized out>)
    at external/mkl_dnn/src/cpu/gemm_x8s8s32x_inner_product.hpp:44
#4  0x00007fc22ee367cc in mkldnn::inner_product_forward::inner_product_forward (this=0x4f11990,
    aprimitive_desc=..., src=..., weights=..., bias=..., dst=...) at external/mkl_dnn/include/mkldnn.hpp:2813
#5  0x00007fc22ee390bb in tensorflow::MklIPFwdPrimitive<float, Eigen::QUInt8, Eigen::QInt8, Eigen::QInt32, Eigen::QUInt8>::Setup (this=this@entry=0x38df300, IPFwdDims=...) at tensorflow/core/kernels/mkl_qmatmul_op.cc:267
#6  0x00007fc22ee3965b in tensorflow::MklIPFwdPrimitive<float, Eigen::QUInt8, Eigen::QInt8, Eigen::QInt32, Eigen::QUInt8>::MklIPFwdPrimitive (this=0x38df300, IPFwdDims=...) at tensorflow/core/kernels/mkl_qmatmul_op.cc:77
#7  0x00007fc22ee3a48c in Get (do_not_cache=false, IPFwdDims=...) at tensorflow/core/kernels/mkl_qmatmul_op.cc:298
#8  tensorflow::MklIPOp<Eigen::ThreadPoolDevice, Eigen::QUInt8, Eigen::QInt8, Eigen::QInt32, Eigen::QUInt8, Eigen::QUInt8, true>::Compute (this=0x1a69760, context=context@entry=0x7fc20f8b6730)
    at tensorflow/core/kernels/mkl_qmatmul_op.cc:499
#9  0x00007fc22ee40f0b in tensorflow::MklQuantizedIPOp<Eigen::ThreadPoolDevice, Eigen::QInt32, Eigen::QUInt8, Eigen::QUInt8, true>::Compute (this=<optimized out>, context=0x7fc20f8b6730)
    at tensorflow/core/kernels/mkl_qmatmul_op.cc:752
#10 0x00007fc229a1ca4c in tensorflow::(anonymous namespace)::ExecutorState::Process (this=<optimized out>,
    tagged_node=..., scheduled_nsec=0) at tensorflow/core/common_runtime/executor.cc:1817
```
看到对应的tensorflow的kernel是MklIPFwdPrimitive

## wnd数据集
google搜索criteo-kaggle
Display Advertising Challenge
https://blog.csdn.net/horizonheart/article/details/78891501
https://www.kaggle.com/c/criteo-display-ad-challenge
