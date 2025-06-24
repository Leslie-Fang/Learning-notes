---
title: tensorflow二次开发
date: 2019-02-27 10:50:46
tags: 技术 tensorflow
---
## 编译
* 方法1：
```
./configure
bazel build --config=opt //tensorflow/tools/pip_package:build_pip_package
build出错清理：
/root/.cache/bazel
把下面的之前出错的缓存文件给删除掉
生成whell包
bazel-bin/tensorflow/tools/pip_package/build_pip_package /root/tensorflow/wheel_pkg/build_withSource
```

* 方法2：
```
yes "" | python configure.py
bazel build --config=mkl -c opt --copt=-O3 -s //tensorflow/tools/pip_package:build_pip_package
生成whell包
bazel-bin/tensorflow/tools/pip_package/build_pip_package /root/tensorflow/wheel_pkg/build_withSource
```

### 编译命令和过程分析
视频：https://www.youtube.com/watch?v=Rw-KrbfyABQ
https://www.cnblogs.com/shouhuxianjian/p/9416934.html

运行configure.py会把一些编译参数放入.bazelrc和.tf_configure.bazelrc文件里面(https://www.jianshu.com/p/5cd111ebb8bb)
bazelrc文件的解释
https://docs.bazel.build/versions/master/guide.html

build 后面接的都是默认的编译参数
build:mkl 后面接的编译参数只有当bazel build --config=mkl的时候mkl后面的编译参数才会起作用

**-c的选项有可能是--config的缩写**

bazel build的其他编译选项：
https://docs.bazel.build/versions/master/user-manual.html
--copt： This option takes an argument which is to be passed to the compiler. 所以--copt后面传进来的都是gcc或者是icc的编译参数

--strip是否删除debug信息，never表示不删除debug信息

### 增量编译
直接bazel build
然后重新生成wheel包
pip unistall tensorflow
**一定先卸载然后重新安装**
否则还是原来的包

## 编译之后
生成pywrap_tensorflow_internal.py 以及 pywrap_tensorflow_internal.cc在~/.cache/bazel目录下面,所有代码都在\_pywrap_tensorflow_internal.so 的动态链接库里面
pywrap_tensorflow_internal.py: 负责对接上层 Python 调用
pywrap_tensorflow_internal.cc: 负责对接下层 C API 调用

* pywrap_tensorflow_internal.py 模块首次被导入时，自动地加
载 \_pywrap_tensorflow_internal.so 的动态链接库；其中， \_pywrap_tensorflow_internal.so
包含了整个 TensorFlow 运行时的所有符号。
* 在 pywrap_tensorflow_internal.cc 的实现中，静态注册了一个函数符号表，实现了 Python 函数名到 C 函数名的二元关系。在运行时，按照 Python 的函数名称，匹找到对应的 C 函数实现，最终实现 Python 到 c_api.c 具体实现的调用关系。

## 调整tensorflow运行的日志等级
TF代码又两个函数打印日志,LOG以及VLOG
LOG是正常的打印日志，通过TF_CPP_MIN_LOG_LEVEL
```
export TF_CPP_MIN_LOG_LEVEL=level
```
去设置，值越小，打印日志越多
VLOG通过
```
export TF_CPP_MIN_VLOG_LEVEL=level
```
去设置，但是VLOG只有在LOG等级为0的时候设置才有用
比如要打印mkl_layout_pass.cc初始化rewirte op时的信息
```
export TF_CPP_MIN_LOG_LEVEL=0
export TF_CPP_MIN_VLOG_LEVEL=1
```

## 编译debug版本的tensorflow
添加 -c dbg选项
移除优化选项 --copt=-O3 以及 -c opt
```
bazel build --config=mkl -s -c dbg //tensorflow/tools/pip_package:build_pip_package
```

debug版本编译完大概有20G左右
export OMP_NUM_THREADS=1
设置intra和inter值为1

### 指定编译目录
默认编译在/root/.cache/bazel目录下面，也可以自己指定编译目录
```
build_dir=/home/bazel_build
bazel --output_user_root=$build_dir clean
bazel --output_user_root=$build_dir build --config=mkl --copt=-mavx2 --copt=-O3 -s -c dbg //tensorflow/tools/pip_package:build_pip_package
```
### 编译报错找不到--march=broadwell
使用gcc6.3以及以上版本，低版本的编译器不认识broadwell的选项

### whell太大无法打包
https://github.com/tensorflow/tensorflow/issues/5538

## 替换mkldnn版本
以TF从0.18升级到0.19为例
### 下载mkldnn0.19计算sha256sum
```
wget https://github.com/intel/mkl-dnn/archive/v0.19.tar.gz
sha256sum v0.19.tar.gz
记录这个结果
ba39da6adb263df05c4ca2a120295641fc97be75b588922e4274cb628dbe1dcd
后面会用到
```

### 修改$tensorflow_root/tensorflow/workspace.bzl
搜索mkl_dnn
```
121     # Important: If you are upgrading MKL-DNN, then update the version numbers
 122     # in third_party/mkl_dnn/mkldnn.BUILD. In addition, the new version of
 123     # MKL-DNN might require upgrading MKL ML libraries also. If they need to be
 124     # upgraded then update the version numbers on all three versions above
 125     # (Linux, Mac, Windows).
 126     tf_http_archive(
 127         name = "mkl_dnn",
 128         build_file = clean_dep("//third_party/mkl_dnn:mkldnn.BUILD"),
 129         sha256 = "38a1c02104ee9f630c1ad68164119cd58ad0aaf59e04ccbe7bd5781add7bfbea",
 130         strip_prefix = "mkl-dnn-0.18",
 131         urls = [
 132             "http://mirror.tensorflow.org/github.com/intel/mkl-dnn/archive/v0.18.tar.gz",
 133             "https://github.com/intel/mkl-dnn/archive/v0.18.tar.gz",
 134         ],
 135     )
```
* 把里面所有0.18替换成0.19
* 替换上面得到的sha256sum

### 看第二步的注释和代码
需要修改"//third_party/mkl_dnn:mkldnn.BUILD"
$tensorflow_root/tensorflow/workspace.bzl
vim $tensorflow_root/third_party/mkl_dnn/mkldnn.BUILD
把里面的版本号从0.18改到0.19

注意：
tensorflow里面，mkldnn是被当做source code编译进去的，
所以不存在动态链接库

check:
build_dir/b3a4cb07d89ceca0353d37b5d32ffadc/external/mkl_dnn
里面是mkldnn下载下来的代码
里面有个readme文件在开头的地方可以check版本是0.18还是0.19

## gdb 调试
二种方法方法去debug TF:
method1:
```
1. gdb python
2. run file.py
3. bt
```
method2:
```
1. 跑测试
2. top 看到python进程的pid
3. gdb -p pid
挂上之后，原来测试会挂住
break 函数名或者其它打上断点,tensorflow找不到符号的情况下可以 文件名:line的方式去打断点
continue 继续测试直到core-dump
```
**如何添加python的信息** 参考这个blog
http://jcf94.com/2018/01/13/2018-01-13-tfunpacking/

### warning找不到文件
dir 目录
去指定文件的搜索根目录
使用gdbgui去调试的时候，也需要指定了目录之后才可以显示文件

### 调试前的参数设置以及技巧
**所有并行计算线程设置为1**，避免多线程导致断点带来的麻烦
命令后加&echo $! 输出PID，进行gdb -p的调试

## mkldnn调试
```
export MKLDNN_VERBOSE=1
python ***
```
在运行测试之前，添加环境变量
可以打出mkldnn的信息
每一行的信息Each line with verbose information is formatted as a comma-separated list containing:
* mkldnn_verbose
* stage, e.g. create or exec
* primitive-kind, e.g. convolution, reorder, sum, ...
* primitive implementation name
* propagation-kind, e.g. forward_training
* input/output data info, e.g. data type and data format
* auxiliary information, e.g. algorithm or number of input
* problem description
  * for convolution the problem description is dumped in benchdnn friendly format
  * for reorder, sum, and concat problem description is simply logical dims
  * for other primitives the problem description is similar to convolution one
* execution time in milliseconds

<!--more-->
## 看python到C++调用关系
### 以Session 为例子：tf.Session时候的调用关系
* python api
/root/tensorflow_src/test_code/private-tensorflow/tensorflow/python
目录下面：
1. grep -rni "class Session"
client/session.py:1475:class Session(BaseSession):
里面调用了baseSession的构造函数
2. 看baseSession
里面调用了tf_session
```
self._session = tf_session.TF_NewSessionRef(self._graph._c_graph, opts)
from tensorflow.python import pywrap_tensorflow as tf_session
```

3. 看pywrap_tensorflow.py
这个就是对应了编译出来的so文件

4. 在source insight里面搜索TF_NewSessionRef
看到定义在tf_session_help.cc里面
里面调用了TF_NewSession

5. source insight里面搜索TF_NewSession
已经进入到C++ 代码内部

### 以matmul为列
https://ggaaooppeenngg.github.io/zh-CN/2018/05/29/Tensorflow-%E7%9A%84-Tensor-%E5%92%8C-OpKernel-%E5%88%86%E6%9E%90/
调用 tf.matmul(a,b)
1. 查看
```
grep -rni "tf_export.*matmul" #这个函数需要用tf_export导出
```
ops/math_ops.py:2277:@tf_export("linalg.matmul", "matmul")

2. 看math_ops.py:2277
api的使用有详细的解释
调用了gen_math_ops.batch_mat_mul 或者 gen_math_ops.mat_mul

3. 看gen_math_ops.py
```
find / -name "gen_math_ops.py"
```
这个文件看文件名字，应该是在编译的时候生成的
这个文件里面搜:batch_mat_mul

4. batch_mat_mul函数
这个函数里面调用了
```
_result = _pywrap_tensorflow.TFE_Py_FastPathExecute(
        _ctx._context_handle, _ctx._eager_context.device_name, "BatchMatMul",
        name, _ctx._post_execution_callbacks, x, y, "adj_x", adj_x, "adj_y",
        adj_y)
```
所以C++里面的op函数应该是BatchMatMul

5. 搜索所有注册这个op的地方
搜索op定义
```
[root@localhost private-tensorflow]# grep -rni "REGISTER_OP(\"MatMul\")"
tensorflow/core/ops/math_ops.cc:763:REGISTER_OP("MatMul")
```

搜索op的kernel实现
```
grep -rni "Name(\"MatMul\")"
```
找到所有定义operation
break 文件名:行
在每个computer的d地方打断点
看看调用到了哪个kernel

看class MatMulOp 的Compute方法里面最后调用了LaunchMatMul方法
LaunchMatMul 继承自LaunchMatMulBase，在 LaunchMatMulBase 当中调用了 functor::MatMulFunctor，这个 functor 主要就会执行乘法操作

MatMulFunctor里面调用了MatMul方法
MatMul方法里面进一步调用了out.device(d) = in0.contract(in1, dim_pair);

contract是Eigen的一个方法，表示矩阵相乘，Eigen是一套高效的C++中调用的数学平台，里面实现了很多通用的数学运算。


### 以conv2d为例
这个人博客很多好文章：http://lanhin.xyz/
http://lanhin.xyz/2018/10/29/tensorflow%E4%B8%AD2d%E5%8D%B7%E7%A7%AF%E4%BB%A3%E7%A0%81%E7%AE%80%E6%9E%90/
1. python 接口 tf.nn.conv2d
```
grep -rni "tf_export.*conv2d"
```
tensorflow_src/test_code/private-tensorflow/tensorflow/python/ops/nn_ops.py:1376:@tf_export("nn.conv2d", v1=[])

2. 查找输出的地方
```
find / -name "gen_math_ops.py"
```

3. 查看op注册和实现的地方
```
grep -rni "REGISTER_OP(\"Conv2D\")"
grep -rni "Name(\"Conv2D\")"
```

4. 进入conv_ops.cc文件
看Compute方法

输入为浮点数float调用LaunchDeepConvOp<Device, T>::Run

其它输入类型调用launcher_
进一步看调用到了
LaunchConv2DOp<CPUDevice, T>::operator()
再往下
tensorflow::LaunchGeneric::operator
这个函数里面通过不同的条件判断调用两个不同的计算kernel：functor::MatMulConvFunctor<Device, T>()和functor::SpatialConvolution<Device, T>()

MatMulConvFunctor定义在conv_2d.h文件里面
out.device(d) = in0.contract(in1, dim_pair, output_kernel);
到最后还是调用了矩阵乘法的函数
这个contract应该是eigen库提供的接口

### INT8 operation
1. 读取RN50 int8的pb
用tensorboard查看
看到用到了op：QuantizedConv2DWithBiasAndReluAndRequantize

**搜索不到对应op的时候**
tensorflow做了op的转换
private-tensorflow\tensorflow\core\graph\mkl_layout_pass.cc
参考这个文件
果然再这个文件里面可以搜索到
QuantizedConv2DWithBiasAndReluAndRequantize
mkl_layout_pass.cc 根据PPT里面的解释，会把标准的输入的TF的graph转换成mkl优化的图，里面有个run函数应该是转换的入口


也有可能定义tensorflow/core/api_def/base_api/api_def_QuantizedMatMulWithBias.pbtxt
这个目录下面也可能定义了pb文件

python api有两种定义方法（https://groups.google.com/a/tensorflow.org/forum/#!topic/developers/LmKn-y7LZ_E）：
Python API endpoints are currently added using 2 ways:
1. api_def_*.pbtxt files (python_op_gen_internal.cc would actually add tf_export decorator for each *visible* endpoint specified in api_def_*.pbtxt files)
2. tf_export decorators

2. 搜索这个op
```
[root@localhost ~]# grep -rni "name(\"QuantizedConv2DWithBiasAndReluAndRequantize\")"
tensorflow_src/test_code/private-tensorflow/tensorflow/core/kernels/mkl_conv_ops.cc:1997:REGISTER_KERNEL_BUILDER(Name("QuantizedConv2DWithBiasAndReluAndRequantize")
```

这个op对应的kernel实现就是QuantizedConv2DWithBiasAndReluAndRequantize
对应的kernel叫做NoOp
看到注释：
**// Register NoOp kernel for QuantizedConv2DWithBiasAndRelu to get a python
// interface.
// This kernel will be replaced by an MKL kernel during graph-optimization pass.**

NoOp是因为这个op在图优化阶段被rewrite了(mkl_layout_pass.cc的RunPass函数)

同一个文件里面看另外一个op
```
_MklQuantizedConv2DWithBiasSumAndRelu
```
对应的kernel是MklQuantizedConv2DSumReluOp
继承了MklQuantizedConv2DOp这个kernel
MklQuantizedConv2DOp这个kernel继承了MklConvOp
MklQuantizedConv2DOp的compute方法首先调用了
```
// Compute int32 output tensor
MklConvOp<Device, quint8, qint8, Tbias, Toutput, Ttemp_output, int32,
          biasEnabled, false>::Compute(context);
```
MklConvOp里面的compute方法调用了mkldnn
conv_fwd->Execute执行mkldnn的计算

**注意**
class MklConvOp在这个文件里面有两个类的定义
通过template <typename Device,
在创建对象时传入的参数可以区分创建了哪个类
一个类使用了mkl，调用dnnExecute_F32的方法
另一个类使用了mkldnn调用conv_fwd->Execute

根据文件里面的宏的定义，应该只有一个函数会被编译出来

看这个mkldnn的类的实现代码，可以先看看MKLDNN的[教程](!https://intel.github.io/mkl-dnn/index.html)和实例代码mkldnn代码库的simple_net.cpp以及[解释](!https://intel.github.io/mkl-dnn/ex_simplenet.html)
基本概念比较清晰，先创建memory/operator descriptor,再创建对应的Primitive descriptor ，最后创建primitive,然后把primitive放到stream里面去执行
tensorflow的这个类的实现follow这个逻辑只是加了一些封装
至于mkldnn里面进一步的实现(如何多线程等)就是mkldnn的事情了
可以看我的mkldnn的文章

## 自己定义个operation
参考文档：http://wiki.jikexueyuan.com/project/tensorflow-zh/how_tos/adding_an_op.html#AUTOGENERATED-adding-a-new-op

### 定义operation
```
#include "tensorflow/core/framework/op.h"
REGISTER_OP("ZeroOut")
    .Input("to_zero: int32")
    .Output("zeroed: int32");
```

### 定义kernel
```
#include "tensorflow/core/framework/op_kernel.h"
using namespace tensorflow;

class ZeroOutOp : public OpKernel {
 public:
  explicit ZeroOutOp(OpKernelConstruction* context) : OpKernel(context) {}
  void Compute(OpKernelContext* context) override {
    // 获取输入 tensor.
    const Tensor& input_tensor = context->input(0);
    auto input = input_tensor.flat<int32>();
   // 创建一个输出 tensor.
    Tensor* output_tensor = NULL;
    OP_REQUIRES_OK(context, context->allocate_output(0, input_tensor.shape(),
                                                     &output_tensor));
    auto output = output_tensor->template flat<int32>();
    // 设置 tensor 除第一个之外的元素均设为 0.
    const int N = input.size();
    for (int i = 1; i < N; i++) {
      output(i) = 0;
    }
    // 尽可能地保留第一个元素的值.
    if (N > 0) output(0) = input(0);
  }
};
REGISTER_KERNEL_BUILDER(Name("ZeroOut").Device(DEVICE_CPU), ZeroOutOp);
```

### 添加python wrap
经过前面两步在编译之后，可以在bazel-genfiles/tensorflow/python/ops/gen_user_ops.py文件，比如我的一个例子
vim /home/lesliefang/bazel_build/615e7e34d0a05b2b7ebac45eda8ba3c5/execroot/org_tensorflow/bazel-out/k8-opt/bin/tensorflow/tools/pip_package/build_pip_package.runfiles/org_tensorflow/tensorflow/python/ops/gen_user_ops.py
里面找到对应的operation的函数
为了使得python可以调用到,在tensorflow/python/user_ops/user_ops.py 文件中添加接口
```
@tf_export(v1=['user_ops.leslie_zero_out'])
def leslie_zero_out(input):
  """Example of overriding the generated code for an Op."""
  return _gen_user_ops.zero_out(input)
```

### 测试
重新编译之后安装之后
测试代码
```
import tensorflow as tf
import numpy as np
import datetime
import os
import time

if __name__ == "__main__":
	#time.sleep(30)
	with tf.Session() as sess:
		sess.run(tf.global_variables_initializer())
		result = tf.user_ops.leslie_zero_out([5, 4, 3, 2, 1])
		print("result is {}".format(result))
		print("result is {}".format(sess.run(result)))
```

## 多线程
To write a multi-threaded CPU kernel, the Shard function in **work_sharder.h** can be used. This function shards a computation function across the threads configured to be used for intra-op threading (see intra_op_parallelism_threads in config.proto).


## 核心运行机制
推荐一个很好的Blog:http://jcf94.com/2018/01/13/2018-01-13-tfunpacking/
这个blog对C++部分session的机制分析的很清楚

这边从python调用session.run开始分析
### 在python里面
1.
session.run
2.  
```
      result = self._run(None, fetches, feed_dict, options_ptr,
                         run_metadata_ptr)
```
3. 在_run里面
 ```
 results = self._do_run(handle, final_targets, final_fetches,
                       feed_dict_tensor, options, run_metadata)
 ```
 4. do_run里面
```
return self._call_tf_sessionrun(
    options, feed_dict, fetch_list, target_list, run_metadata)
```

 5. call_tf_sessionrun里面
 ```
 return tf_session.TF_SessionRun_wrapper(
    self._session, options, feed_dict, fetch_list, target_list,
    run_metadata)
 ```

TF_SessionRun_wrapper 定义在pywrap_tensorflow_internal.py里面
就是python和C++的桥梁

### 下面进入C++的部分
1. TF_SessionRun_wrapper_helper函数
里面调用了TF_SessionRun

2. TF_SessionRun 函数
调用了TF_Run_Helper函数

3. TF_Run_Helper函数
调用了session->Run函数

4. 这是个虚函数
用gdb跟进去看
参考这篇文章：https://zhuanlan.zhihu.com/p/26031658
local用direction_session
分布式用grpc_session
所以我们这边调用到了DirectSession::Run

5. 看DirectSession::Run函数
这个函数的分析：http://jcf94.com/2018/01/13/2018-01-13-tfunpacking/
* GetOrCreateExecutors函数里面会去寻找有没有符合条件的exectuor，不存在的话则调用CreateExecutors函数去创建executors
同时CreateExecutors里面调用到了CreateGraphs
在CreateExecutors调用了CreateGraphs之后看到：
```
params.create_kernel = [this, lib, opseg](const NodeDef& ndef,
                                              OpKernel** kernel)
```
我理解就是在这里实现了param里面的创建kernel的函数指针
在CreateExecutors的最后调用了NewExecutor函数，会传入param变量(里面带上了create_kernel方法)
NewExecutor函数里面通过工厂模式来生成Executor
是个虚函数，通过gdb看到里面调用了
tensorflow::(anonymous namespace)::DefaultExecutorRegistrar::Factory::NewExecutor (this=0x1fffd10, params=..., graph=...,
    out_executor=0x72fdee8) at tensorflow/core/common_runtime/executor.cc:2857
```
class Factory : public ExecutorFactory {
  Status NewExecutor(const LocalExecutorParams& params,
                     std::unique_ptr<const Graph> graph,
                     std::unique_ptr<Executor>* out_executor) override {
    Executor* ret = nullptr;
    TF_RETURN_IF_ERROR(NewLocalExecutor(params, std::move(graph), &ret));
    out_executor->reset(ret);
    return Status::OK();
  }
};
```
里面调用了NewLocalExecutor
进一步调用ExecutorImpl->Initialize函数
这个函数里面调用了params_.create_kernel函数去创建kernel
(这个create_kernel函数就是之前在CreateExecutors函数里面定义的)
同时在这个函数里面看到了一行注释
```
// Preprocess every node in the graph to create an instance of op
// kernel for each node.
```
### 调试CreateExecutors的create_kernel函数
gdb断点进去CreateKernel函数
tensorflow/core/common_runtime/function.cc:521
调用到526行的CreateKernel函数
tensorflow/core/common_runtime/function.cc:526
executor.cc的CreateNonCachedKernel函数
op_kernel.cc的CreateOpKernel函数（\*kernel = registration->factory->Create(&context);）
mkl_conv_ops.cc的TF_CALL_float(REGISTER_MKL_CPU_2D_FUSED);函数
mkl_conv_ops.cc的MklFusedConvOp的构造函数


所以调用session.run多次，因为已经存在符合条件的exectuors，并不会多次创建图
（别人的评论：第一次执行 sess.run(....) 的时候会根据 python 层的图构造出 C++ 层的图然后保存下来，之后如果下次 sess.run() 的目标节点是相同的，就不需要重新构造一遍了。详细可以去分析 sess.run() 的执行流程）

* 调用到了RunInternal函数

6. RunInternal函数
里面调用了item.executor->RunAsync(args, barrier->Get());
去执行异步计算

7. 通过日志知道RunAsync会调用到executor的Process()函数
process函数做了什么：
http://jcf94.com/2018/01/13/2018-01-13-tfunpacking/
遍历每个节点，针对每个节点的kernel进行计算（调用device->Compute，里面调用op_kernel->Compute(context);）
在每个kernel里面都可以搜索到对应的Compute函数

## 看一个inner product的kernel是怎么生成的
断点打在
```
b mkl_qmatmul_op.cc:183(一个setup函数里面)
```
汾西代码知道这个setup函数是设置上下文变量的
查看调用栈
```
#0  tensorflow::MklIPFwdPrimitive<float, Eigen::QUInt8, Eigen::QInt8, Eigen::QInt32, Eigen::QUInt8>::Setup (this=0x3d1a300, IPFwdDims=...)
    at tensorflow/core/kernels/mkl_qmatmul_op.cc:183
#1  0x00007f6a77ee938c in tensorflow::MklIPFwdPrimitive<float, Eigen::QUInt8, Eigen::QInt8, Eigen::QInt32, Eigen::QUInt8>::MklIPFwdPrimitive (this=0x3d1a300, IPFwdDims=...)
    at tensorflow/core/kernels/mkl_qmatmul_op.cc:77
#2  0x00007f6a77ee81c3 in tensorflow::MklIPFwdPrimitiveFactory<float, Eigen::QUInt8, Eigen::QInt8, Eigen::QInt32, Eigen::QUInt8>::Get (IPFwdDims=..., do_not_cache=false)
    at tensorflow/core/kernels/mkl_qmatmul_op.cc:298
#3  0x00007f6a77ee0515 in tensorflow::MklIPOp<Eigen::ThreadPoolDevice, Eigen::QUInt8, Eigen::QInt8, Eigen::QInt32, Eigen::QUInt8, Eigen::QUInt8, true>::Compute (
    this=0x1ea0f20, context=0x7f6a53f1d5f0) at tensorflow/core/kernels/mkl_qmatmul_op.cc:499
#4  0x00007f6a77edee0e in tensorflow::MklQuantizedIPOp<Eigen::ThreadPoolDevice, Eigen::QInt32, Eigen::QUInt8, Eigen::QUInt8, true>::Compute (this=0x1ea0f20,
    context=0x7f6a53f1d5f0) at tensorflow/core/kernels/mkl_qmatmul_op.cc:752
#5  0x00007f6a78410eae in tensorflow::Device::Compute (this=0x40a6780, op_kernel=0x1ea0f20, context=0x7f6a53f1d5f0) at ./tensorflow/core/common_runtime/device.h:89
#6  0x00007f6a6c90f868 in tensorflow::(anonymous namespace)::ExecutorState::Process (this=0x54f6480, tagged_node=..., scheduled_nsec=0)
    at tensorflow/core/common_runtime/executor.cc:1817
```
* \#0 mkl_qmatmul_op.cc:183 在tensorflow里面这个primitive的setup函数
看这个setup里面，看到先创建mkldnn的primitive的desc
```
// create a inner product
 context_.fwd_desc.reset(new inner_product_forward::desc(
       prop_kind::forward_inference, *context_.src_md, *context_.weight_md,
       *context_.bias_md,
       *context_.dst_md));
```
然后通过这个desc去创建primitive_desc(pd),跟进到mkldnn里面看，就是在创建pd的时候回去遍历mkldnn里面所有pd找到对应的满足条件的pd
* \#1 mkl_qmatmul_op.cc:77 MklIPFwdPrimitive的构造函数
* \#2 mkl_qmatmul_op.cc:298 MklIPFwdPrimitiveFactory的Get函数，Get函数根据输入的MklIPFwdParams去try to find a suitable one in pool
没有找到的话(if (IP_fwd == nullptr))会去创建
* \#3 mkl_qmatmul_op.cc:499 MklIPOp的compute方法，里面调用了MklIPFwdPrimitiveFactory的Get方法去拿到对应的IP_fwd(Primitive)
MklIPOp的compute方法 应该是tensorflow在运行图的节点的时候会被调用到的方法
**继续看这个MklIPOp的compute方法**
后面会调用IP_fwd->Execute(src_data, weight_data, bias_data, dst_data);
去做计算
这个根据前几步选中的mkldnn的pd，会调用到mkldnn的submit函数(context_.fwd_stream->submit(context_.fwd_primitives);)
可以用GDB去跟进mkldnn去看调用关系，这里已经比较好理解了
**结论**
所以tensorflow的node到mkldnn的kernel的对应关系，是在第一次运行这个图的时候确认的，同时如果set了cache(默认都是设置的),后面几次运行的时候就会保留这个对应关系
* \#4 mkl_qmatmul_op.cc:752 MklQuantizedIPOp的Compute函数，这个函数会去调用MklIPOp的compute方法
* \#5 device.h:89 Device的Compute()是个虚函数,对应了device信息
* \#6 executor.cc:1817 ExecutorState::Process函数，这里已经是tensorflow创建了exectuor之后的执行了
* \#7 executor.cc:2258  ExecutorState::ScheduleReady

**总结**，关键是这个MklIPOp的compute方法，先通过Get方法去获得对应的mkldnn的kernel，然后调用execute去执行

## 通过pb文件去看调用的kernel
### 读取pb文件，查看模型的结构
使用tensorboard或者Netron
推荐使用Netron，很好用，里面还可以看到各个节点的参数的值

### 打印pb文件中每个节点的名字
* 在代码里面加载输出每个节点的名字
```
graph_def = graph_pb2.GraphDef()
with open(args.input_graph, "rb") as f:
  graph_def.ParseFromString(f.read()) #f就是pb文件
for node in graph_def.node:
    k = node.name
    print("node op is {}".format(node.op))
```
打印出node的名字
比如其中一个MatMul
* 加载pb用tensorboard大概看一下
```
2 import pandas as pd
3 import csv
4 import struct
5 from PIL import Image
6 import numpy as np
7 import datetime
8 import os
9 import argparse
10 import tensorflow as tf
11
12 if __name__ == "__main__":
13         parser = argparse.ArgumentParser()
14         parser.add_argument("mode", help="display a square of a given number")
15         args = parser.parse_args()
16         from tensorflow.python.platform import gfile
17         with gfile.FastGFile(args.mode, 'rb') as f:
18                 graph_def = tf.GraphDef()
19                 graph_def.ParseFromString(f.read())
20                 for node in graph_def.node:
21                         print("node name is: {} \t node op is: {}".format(node.name,node.op))
22                 #tensorboard
23                 with tf.Session() as sess:
24                         sess.graph.as_default()
25                         tf.import_graph_def(graph_def, name='')
26                         summaryWriter = tf.summary.FileWriter('log/', sess.graph)
```
跑完之后，命令行运行
tensorboard --logdir log/

* 在tensorlfow里面搜索注册这个op和kernel的地方
比如第二步打印看到的node.op是 Conv2D
在代码里面搜索
```
grep -rni "Name(\".*Conv2D.*\")"
```
因为注册的kernel可能是Conv2D
也有可能加了mkl前缀比如:REGISTER_KERNEL_BUILDER(Name("\_MklConv2D")
在directSession，创建新的exector的时候会去优化graph，这个时候会把Conv2D这个op转换成_MklConv2D，一般就是添加_MKL的前缀
在mkl_layout_pass.cc这个文件的RunPass函数里面，会去做图的优化，包括临近节点的合成，op的rewrite以及mkldnn节点前添加数据格式的转换等op
**创建kernel时候的调用栈**
断点打在mkl_conv_ops.cc:861
```
#0  tensorflow::MklConvOp<Eigen::ThreadPoolDevice, float, float, float, float, float, int, false, false>::MklConvOp (this=this@entry=0x36b35400,
    context=context@entry=0x7ffca8d435c0) at tensorflow/core/kernels/mkl_conv_ops.cc:861
#1  0x00007fa3b9de7ecc in tensorflow::MklFusedConvOp<Eigen::ThreadPoolDevice, float, float, float, float, float, int, true>::MklFusedConvOp (
    this=0x36b35400, context=0x7ffca8d435c0) at tensorflow/core/kernels/mkl_conv_ops.cc:1474
#2  0x00007fa3b9dcd7b2 in operator() (__closure=0x0, context=0x7ffca8d435c0) at tensorflow/core/kernels/mkl_conv_ops.cc:2165
#3  tensorflow::<lambda(tensorflow::OpKernelConstruction*)>::_FUN(tensorflow::OpKernelConstruction *) ()
    at tensorflow/core/kernels/mkl_conv_ops.cc:2165
#4  0x00007fa3b469ac77 in tensorflow::CreateOpKernel (device_type=..., device=device@entry=0x3c346e0, allocator=allocator@entry=0x1c1e380,
    flib=flib@entry=0x36bae2c0, node_def=..., graph_def_version=0, kernel=0x15b5c4bc8) at tensorflow/core/framework/op_kernel.cc:1302
#5  0x00007fa3b498f80f in tensorflow::CreateNonCachedKernel (device=0x3c346e0, flib=flib@entry=0x36bae2c0, ndef=...,
    graph_def_version=<optimized out>, kernel=kernel@entry=0x15b5c4bc8) at tensorflow/core/common_runtime/executor.cc:2764
#6  0x00007fa3b49aaaf7 in tensorflow::FunctionLibraryRuntimeImpl::CreateKernel (this=0x36bae2c0, ndef=..., lib_def=0x372c000, kernel=0x15b5c4bc8)
    at tensorflow/core/common_runtime/function.cc:539
#7  0x00007fa3b49aac18 in tensorflow::FunctionLibraryRuntimeImpl::CreateKernel (this=<optimized out>, ndef=..., kernel=<optimized out>)
    at tensorflow/core/common_runtime/function.cc:515
#8  0x00007fa3ba11e40b in operator() (kernel=0x15b5c4bc8, ndef=..., __closure=0x2ef1e660) at tensorflow/core/common_runtime/direct_session.cc:1261
#9  std::_Function_handler<tensorflow::Status(const tensorflow::NodeDef&, tensorflow::OpKernel**), tensorflow::DirectSession::CreateExecutors(const tensorflow::CallableOptions&, std::unique_ptr<tensorflow::DirectSession::ExecutorsAndKeys>*, std::unique_ptr<tensorflow::DirectSession::FunctionInfo>*, tensorflow::DirectSession::RunStateArgs*)::<lambda(const tensorflow::NodeDef&, tensorflow::OpKernel**)> >::_M_invoke(const std::_Any_data &, const tensorflow::NodeDef &, <unknown type in /home/lesliefang/venv_python36_RN50_Debug/lib/python3.6/site-packages/tensorflow/python/_pywrap_tensorflow_internal.so, CU 0x23b51f7a, DIE 0x23c57fc7>) (__functor=..., __args#0=..., __args#1=<optimized out>)
    at /home/lesliefang/gcc63/lib/gcc/x86_64-pc-linux-gnu/6.3.0/../../../../include/c++/6.3.0/functional:1717
#10 0x00007fa3b49a164e in operator() (__args#1=<optimized out>, __args#0=..., this=0x169d87cf8)
    at /home/lesliefang/gcc63/lib/gcc/x86_64-pc-linux-gnu/6.3.0/../../../../include/c++/6.3.0/functional:2127
#11 tensorflow::(anonymous namespace)::ExecutorImpl::Initialize (this=this@entry=0x169d87ce0) at tensorflow/core/common_runtime/executor.cc:620
#12 0x00007fa3b49a3646 in tensorflow::NewLocalExecutor (params=..., graph=..., executor=executor@entry=0x7ffca8d44218)
    at tensorflow/core/common_runtime/executor.cc:2749
#13 0x00007fa3b49a36d2 in tensorflow::(anonymous namespace)::DefaultExecutorRegistrar::Factory::NewExecutor (this=<optimized out>, params=...,
    graph=..., out_executor=0x3ab72bb8) at tensorflow/core/common_runtime/executor.cc:2785
#14 0x00007fa3b49a61b2 in tensorflow::NewExecutor (executor_type=..., params=..., graph=..., out_executor=out_executor@entry=0x3ab72bb8)
    at tensorflow/core/common_runtime/executor_factory.cc:82
#15 0x00007fa3ba128ee4 in tensorflow::DirectSession::CreateExecutors (this=this@entry=0x2edd8480, callable_options=...,
    out_executors_and_keys=out_executors_and_keys@entry=0x7ffca8d448a0, out_func_info=out_func_info@entry=0x7ffca8d448b0,
    run_state_args=run_state_args@entry=0x7ffca8d44fb0) at tensorflow/core/common_runtime/direct_session.cc:1296
#16 0x00007fa3ba12a730 in tensorflow::DirectSession::GetOrCreateExecutors (this=this@entry=0x2edd8480, inputs=..., outputs=..., target_nodes=...,
    executors_and_keys=0x7ffca8d44f48, run_state_args=0x7ffca8d44fb0) at tensorflow/core/common_runtime/direct_session.cc:1429
    #17 0x00007fa3ba12b747 in tensorflow::DirectSession::Run (this=<optimized out>, run_options=..., inputs=..., output_names=..., target_nodes=...,
    ---Type <return> to continue, or q <return> to quit---
        outputs=0x7ffca8d45340, run_metadata=0x7ffca8d453a0) at tensorflow/core/common_runtime/direct_session.cc:749
    #18 0x00007fa3b76729f1 in tensorflow::SessionRef::Run (this=0x38d4a5f0, run_options=..., inputs=..., output_tensor_names=...,
        target_node_names=..., outputs=0x7ffca8d45340, run_metadata=0x7ffca8d453a0) at tensorflow/python/client/session_ref.cc:427
    #19 0x00007fa3b78c2d9d in TF_Run_Helper (session=0x38d4a5f0, handle=handle@entry=0x0, run_options=run_options@entry=0x0, input_pairs=...,
        output_tensor_names=..., c_outputs=c_outputs@entry=0x7ffca8d45708, target_oper_names=..., run_metadata=0x0, status=0x2b657788)
        at tensorflow/c/c_api.cc:787
    #20 0x00007fa3b78c3a3a in TF_SessionRun (session=session@entry=0x3b57ef60, run_options=run_options@entry=0x0, inputs=<optimized out>,
        input_values=<optimized out>, ninputs=<optimized out>, outputs=0x36bbfc00, output_values=0x7ffca8d45708, noutputs=1, target_opers=0x0,
        ntargets=0, run_metadata=0x0, status=0x2b657788) at tensorflow/c/c_api.cc:2638
    #21 0x00007fa3b76710df in tensorflow::TF_SessionRun_wrapper_helper (session=0x3b57ef60, handle=handle@entry=0x0, run_options=0x0, inputs=...,
        input_ndarrays=..., outputs=..., targets=..., run_metadata=0x0, out_status=0x2b657788, py_outputs=0x7ffca8d45a50)
        at tensorflow/python/client/tf_session_helper.cc:410
    #22 0x00007fa3b76711b2 in tensorflow::TF_SessionRun_wrapper (session=<optimized out>, run_options=<optimized out>, inputs=..., input_ndarrays=...,
        outputs=..., targets=..., run_metadata=0x0, out_status=0x2b657788, py_outputs=0x7ffca8d45a50)
        at tensorflow/python/client/tf_session_helper.cc:452
    #23 0x00007fa3b760b8d0 in _wrap_TF_SessionRun_wrapper (args=<optimized out>)
        at bazel-out/k8-dbg/bin/tensorflow/python/pywrap_tensorflow_internal.cc:20508

```

关键代码分析：
op_kernel.cc:1302 CreateOpKernel函数

```
// Everything needed for OpKernel construction.
OpKernelConstruction context(
    device_type, device, allocator, &node_def, op_def, flib, inputs,
    input_memory_types, outputs, output_memory_types, graph_def_version, &s);
*kernel = registration->factory->Create(&context);
```
OpKernelConstruction context构造了找寻合适的tensorflow的条件


总结：tensorflow这边node的多态有两层
* 第一层是在tensorflow自己框架的设计上，在session.run的时候，第一次运行创建exectuor的时候进行
* 第二层多态是mkldnn层面上的，在调用op.Compute的方法的时候，第一次调用会去根据输入的数据类型选择并创建正确的mkldnn的pd

## INT8化操作
理论介绍：
https://aidc.gallery.video/detail/videos/all-videos/video/5790616836001/understanding-new-vector-neural-network-instructions-vnni

**重点推荐这篇文章，介绍量化很详细**
https://petewarden.com/2016/05/03/how-to-quantize-neural-networks-with-tensorflow/


基本思想：
* 对于输入的张量
每一个FP32的输入张量，额外通过一个Min Op得到最小值Min，通过一个Max op得到最大值Max。原始FP32张量，和Min以及Max一起过一个quantize的op得到INT8的张量，再过INT8的计算op(POOL,Conv2D)。再将计算结果，和Min以及Max值一起过一个Dequantize的op反量化得到FP32的输出
如果邻近两个节点都是INT8的量化操作，它们之间的反量化和量化操作可以省略
* 对于原来存储的FP32格式的weight以及bias
直接INT8化存储就可以了，存INT8值以及Min以及Max

TF1.10版本
### transform_graph 工具
tensorflow/tools/graph_transforms 目录下面有个readme去介绍怎么做的
包括transform_graph里面每个trainform操作做了什么
**这一步不是必须的**
对原来的FP32的图做一些预处理的操作
每个操作的内容都写在--transforms参数里面，生成一个列表
每一个操作在对应的文件里面通过
```
REGISTER_GRAPH_TRANSFORM("fold_batch_norms", FoldBatchNorms);
```
函数写到transform_registry里面

在主函数里面遍历--transforms的输入列表，从transform_registry里面找到对应操作的函数，执行操作，返回新的graph_def

### quantize_graph.py 脚本
**这一步是必须的**
这个脚本的作用：
1. 是把原来图中的op转换成对应的INT8操作的op，比如conv2D转换成QuantizedConv2DWithBias或者QuantizedConv2DWithBiasAndRelu或者等等等
2. 同时插入量化和反量化计算的节点，额外得到Min，Max 以及quantize和dequantize的op
3. weights的量化操作也是在这一步做的，将FP32的weights值存成INT8的，有个quantize_weight_eightbit函数，将base_name对应的fp32节点换成int8，min,max 3个节点

转换之后多了几个节点：
输入计算节点之前：
* Min：计算输入张量的最小值
* Max：计算输入张量的最大值
* QuantizeV2： 输入FP32，Min，Max计算量化的INT8输出，输入计算节点
输入计算节点之后：计算节点的输出：比如量化卷积计算的输出是 **INT32** 的(MKLDNN x8s8X32的primitive)
* RequantizationRange：因为输出是INT32的，而且量化成INT8的scale不在原始图里面存着，需要再量化一次，通过RequantizationRange去计算INT32张量的最大值和最小值
* Requantize：具体计算INT32输出量化成INT8
* Dequantize：INT8结果反量化成FP32的格式


### 插入log节点，并得到每一层的参数范围

**这一步是必须的**
使用transform_graph 工具 插入log节点
#### Freeze Re-quantization Range
因为量化卷积(mkldnn)输出是INT32的，需要重新量化成INT8，而且量化成INT8的scale不在原始图里面存着，所以通过这一步，做一次inference，记录scala，去需要再量化一次，通过RequantizationRange去计算
如果量化节点的输出已经是INT8的格式(比如Maxpool节点)，就不需要Re-quantization
这一步 **freeze之后就没有RequantizationRange这个节点了** 只保留了量化的scala

* 找到RequantizationRange这个节点，在这个节点后面插入一个Print节点去打印输出数据的范围信息
RequantizationRange节点似乎是跟在Conv2D节点后面的，打印Conv2D的INT32输出的最大值和最小值
* 选取一部分训练数据，进行inference，记录最大和最小值（Print节点会打出来的），保存成min_max.log文件
python Inference.py 2> min_max.log
因为Print节点的输出是error所以用2去重定向就可以了
* 利用min_max.log和transform_graph工具去freeze这个requantization_ranges这个节点，把节点值变成常量，加快运算
**freeze之后就没有RequantizationRange这个节点了**
freeze之后把这个requantization_ranges节点通过2个const(name/frozen_min和name/frozen_max)替换了
tensorflow/tools/graph_transforms 里面的readme有介绍freeze_requantization_ranges这个transform做了什么

#### Freeze max ranges
Max节点一般在量化之前出现，计算输入张量的最大值，用于量化
* 找到Max这个节点，在这个节点后面插入一个Print节点去打印输出数据的范围信息
* 选取一部分训练数据，进行inference，记录最大值（Print节点会打出来的），保存成max.log文件
* 利用max.log文件去freeze Max这个节点((去除Max节点，用大值const的节点去替换name/frozen_max_only)，加速inference的运算
**freeze之后就没有Max这个节点了**

#### Freeze min ranges
Min节点一般在量化之前出现，计算输入张量的最小值，用于量化
* 找到Min这个节点，在这个节点后面插入一个Print节点去打印输出数据的范围信息

* 选取一部分训练数据，进行inference，记录最大值（Print节点会打出来的），保存成min.log文件
* 利用min.log文件去freeze Min这个节点(去除Min节点，用小值const节点替换name/frozen_min_only)，加速inference的运算
**freeze之后就没有Min这个节点了**

通过这几步之后，quantize_graph.py 脚本生成的6个节点，只剩下了三个:
* QuantizeV2： 输入FP32，Min，Max计算量化的INT8输出，输入计算节点
输入计算节点之后：计算节点的输出：比如量化卷积计算的输出是 **INT32** 的(MKLDNN x8s8X32的primitive)
* Requantize：具体计算INT32输出量化成INT8
* Dequantize：INT8结果反量化成FP32的格式

Requantize 又可以和conv合并成一个节点
### 利用transform_graph 工具
**这一步不是必须的**,最好运行下
1. 因为前面几步产生了一些不需要的节点，利用transform_graph 工具再移除一些不必要的节点strip_unused_nodes
2. 将INT8的conv和后面的requantize节点合并:fuse_quantized_conv_and_requantize


### 对比FP32以及INT8模型
#### FP32
model.pb是训练得到的FP32模型
* conv层调用 op是Conv2D， 经过mkl_layer_pass之后对应了TF里面_mklconv这个op，对应了TF的MklConvOp这个kernel，
```
  REGISTER_KERNEL_BUILDER(Name("_MklConv2D")                               \
                              .Device(DEVICE_CPU)                          \
                              .TypeConstraint<T>("T")                      \
                              .Label(mkl_op_registry::kMklOpLabel),        \
                          MklConvOp<CPUDevice, float, float, float, float, \
                                    float, int32, false, false>);
```
根据这个函数去创建MklConvOp对象并调用Compute方法
对应mkldnn里面的jit_avx512_common_convolution_fwd_t这个primitive
* mkldnn verbose的输出：mkldnn_verbose,exec,convolution,jit:avx512_common,forward_training,fsrc:nChw16c fwei:OIhw16i16o fbia:undef fdst:nChw16c,alg:convolution_direct,mb128_ic32oc64_ih14oh14kh5sh1dh0ph2_iw14ow14kw5sw1dw0pw2,*

#### INT8
一步步量化得到的INT8模型是： frozen_int8_model.pb
这个模型有两个卷积运算，第一个卷积运算没有INT8化，第二个卷积运算INT8化了
我们这里关注第二个卷积运算
* conv调用了 op是QuantizedConv2D， 经过mkl_layer_pass之后对应了TF里面_MklQuantizedConv2D这个op,TF的MklQuantizedConv2DOp这个kernel，MklQuantizedConv2DOp的Compute方法先调用了MklConvOp的Compute的方法
虽然也调用了MklConvOp的Compute的方法
但是MklQuantizedConv2DOp这个kernel是通过
```
    MklConvOp<Device, quint8, qint8, Tbias, Toutput, Ttemp_output, int32,
              biasEnabled, false>::Compute(context);
```
去创建 MklConvOp对象并调用Compute方法，和FP32的模板参数类型不一样对应了Tinput, Tfilter以及Toutput
因为模板参数不一样，调用MklConvOp的compute方法的时候对应找到的对应的mkldnn的pd也不一样，所以对应的mkldnn的primitive也不一样
通过看mkldnn的cpu_engine.cpp的cpu_impl_list怀疑对应了mkldnn的jit_avx512_core_x8s8s32x_convolution_fwd_t<u8,s32>
这个primitive
如何证实：
1. 通过gdb去debug，证实了猜想
2. export MKLDNN_JIT_DUMP=1 去看dump出来的bin，里面果然有mkldnn_dump__jit_avx512_core_x8s8s32x_conv_fwd_ker_t.23.bin
jit_avx512_core_x8s8s32x_convolution_fwd_t这个op里面在满足输入条件时会去调用VNNI的指令集VPDPBUSD

反汇编dump出来的bin文件，VPDPBUSD的指令被dump出来了
我们单步调试，看到mkldnn里面jit_avx512_core_x8s8s32x_convolution_fwd_t里面jcp_ver 是 ver_vnni
同时compute_ker函数(jit_avx512_core_x8s8s32x_conv_kernel.cpp)里面的cpmpute部分也的确调用到了vpdpbusd

* mkldnn verbose的输出：
mkldnn_verbose,exec,convolution,jit_int8:avx512_core,forward_training,fsrc:nhwc fwei:OIhw4i16o4i fbia:undef fdst:nhwc,alg:convolution_direct,mb128_ic32oc64_ih14oh14kh5sh1dh0ph2_iw14ow14kw5sw1dw0pw2,*

## GDB打印变量值
* Tensor变量
```
Tensor* out
p *(unsigned long *)(out->buf_->data_)
```
out->buf_->data_ 是数据指针 const *
根据代码里面数据类型，转换成unsigned long * 类型
再*取指针值

* nodedef 和grahdef
都是定义在 tensorflow/core/framework/.proto 文件里面
用DebugString可以打印值
p nodedef->DebugString()
p grahdef->DebugString()

## 编写和触发单元测试
### 编写
在编写单元测试的时候可以参考Netron查看的模型结构

在tensorflow/python/kernel_tests/ 目录下面写在对应的单元测试的文件里面
比如之前写的concat的单元测试，测试是否成功创建concat op
tensorflow/python/kernel_tests/concat_op_test.py
写在这个文件目录下面

### 触发
```
# All tests (for C++ changes).
$bazel test //tensorflow/...

# All Python tests (for Python front-end changes).
bazel --output_user_root=$build_dir test --config=mkl --copt=-O3 //tensorflow/python/...

# 只想运行一个文件
bazel --output_user_root=$build_dir test --config=mkl --copt=-O3 //tensorflow/python/kernel_tests:concat_op_test
```

##  语法检查
###  pylint 检查python语法规范
```
pip install pylint
## rcfile 文件 指定了pylint使用的规则
export rcfile=$tensorflow_root/tensorflow/tools/ci_build/pylintrc
pylint --rcfile=$rcfile $tensorflow_root/tensorflow/python/kernel_tests/concat_op_test.py
```
输出可能有很多语法不规范，选择和自己这个commit相关的不规范语法去修改

### clang-format 检查 C++ 语法规范
```
#ubuntu：
apt-get install clang-format
#centos：
#http://releases.llvm.org/download.html 下载预编译版本，**现在似乎用不了**
wget https://github.com/llvm/llvm-project/releases/download/llvmorg-8.0.1/clang+llvm-8.0.1-powerpc64le-linux-rhel-7.4.tar.xz

clang-format -style=Google mkl_concat_op.cc 2>&1 | tee mkl_concat_op.cc.bk
diff mkl_concat_op.cc mkl_concat_op.cc.bk
```
运行之后会生成一个标准语法的文件版本，和自己的版本的代码对比，修改语法不规范的地方

## Appendix
### Intel优化版本的介绍
https://www.youtube.com/watch?v=VI5vjB6-zNE
