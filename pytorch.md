---
title: pytorch
date: 2021-01-04 19:41:26
tags: 深度学习 Pytorch
---
## Build
```
git clone --recursive *.git

如果要修改已有的checkin
rm -rf third_party
git submodule sync
git submodule update --init --recursive

## Build
python setup.py build
python setup.py install
python setup.py clean --all

生成wheel包
python setup.py bdist_wheel
```

quantized::linear python 接口定义在
torch/nn/quantized/modules/linear.py

### Pytorch 运行单元测试
https://github.com/pytorch/pytorch/tree/master/benchmarks/operator_benchmark
numactl -c 0 -m 0 python -m pt.qlinear_test --omp_num_threads 28 --mkl_num_threads 28

运行FBGEMM的benchmark
FBGEMM的run to run 变化很大，因为它会自动cflush去刷新内存

### Python 和C++之间的接口
#### 基本介绍
pytorch里面有两种方式：
* 使用pybind11
* 直接使用CPython，python.h 以及使用pyobject*，ATEN相关的函数映射(如下面提高的conv2d)是用这种方式实现的

#### conv2d的例子
通过conv2d的例子来看下python和C++之间的接口层：
* torch/nn/modules/conv.py -> nn.Conv2d
调用了F.conv2d
* torch/nn/functional.py -> conv2d
这个函数里面调用到了 torch.conv2d， 这个实现在C++里面
* torch/csrc/autograd/generated/python_torch_functions.cpp -> THPVariable_conv2d
generated 目录下面的文件都是在编译时生成的，python_torch_functions.cpp文件生成过程的细节参考下面的section
这里简单看一下这个文件的做函数映射的内容：
python_torch_functions.cpp的PyMethodDef torch_functions[]里面定义了映射关系，从而将torch.conv2d 映射到了函数THPVariable_conv2d
```
{"conv2d", (PyCFunction)(void(*)(void))THPVariable_conv2d, METH_VARARGS | METH_KEYWORDS | METH_STATIC, NULL},
```
THPVariable_conv2d 调用lambda函数dispatch_conv2d，进一步调用到函数at::conv2d
* aten/src/ATen/native/Convolution.cpp -> at::conv2d
* aten/src/ATen/native/Convolution.cpp -> at::convolution
* aten/src/ATen/native/Convolution.cpp -> at::_convolution
这个函数会将卷积派发到mkldnn或者cuda去直接计算,下面先看mkldnn的部分
* aten/src/ATen/native/mkldnn/Conv.cpp -> at::mkldnn_convolution
* aten/src/ATen/native/mkldnn/Conv.cpp -> at::_mkldnn_conv2d
* ideep/computations.hpp -> compute_impl
**compute_impl 是convolution_forward 这个结构体的成员函数**
看一下compute_impl这个函数的细节，如何创建mkldnn的Primitive和执行计算：
首先是计算一些mkldnn创建PD需要的参数，然后将这些参数传入这个重点函数
```
fetch_or_create_m(comp, key, src_desc, weights_desc, bias_desc, dst_desc_in, strides, dilates,
        padding_l, padding_r, op_attr, std::forward<Ts>(args)...);
```
fetch_or_create_m 定义在ideep/include/ideep/lru_cache.hpp 里面
```
#define fetch_or_create_m(op, key, ...)                 \
    auto it = find(key);                                \
    if (it == end()) { it = create(key, __VA_ARGS__); } \
    auto op = fetch(it);
```
进一步跟进fetch_or_create_m函数，发现调用到了computation_cache::create 方法
```
auto it = t_store().insert(std::make_pair(key,value_t(std::forward<Ts>(args)...)));
```
value_t(std::forward<Ts>(args)...)) 就是进入了创建primitive的函数
value_t是通过computation_cache的模板传入的
```
template <class value_t, size_t capacity = 1024, class key_t = std::string>
class computation_cache {
```

同时注意到，compute_impl是convolution_forward结构体的成员函数，convolution_forward结构体继承了computation 以及computation_cache 结构体
```
struct convolution_forward: public computation,
  public utils::computation_cache<convolution_forward>
```
所以这里的value_t就是convolution_forward
这里通过value_t 去创建了一个convolution_forward的结构体的对象，然后存入到primitive cache里面，方便后面再次调用，避免重复创建

* ideep/computations.hpp::convolution_forward 的构造函数 convolution_forward
convolution_forward 结构体继承自computation结构体，继承自primitive_group结构体，继承自c_wrapper<mkldnn_primitive_t>
这个构造函数，做了两件事情
```
descriptor forward_descriptor(src_desc, std::forward<Ts>(args)...); #descriptor 继承自primitive descriptor,这里创建了对应的pd
computation::init(forward_descriptor); #这里通过PD去创建primitive
```

#### 细节python_torch_functions.cpp
来具体看下python_torch_functions.cpp 这个文件是如何生成的：
**主要做的事情就是读取yaml文件的内容，去替换template文件对应的部分，生成新的文件**

build的时候会去执行这个脚本tools/setup_helpers/generate_code.py
```
build/build.ninja:41358:  COMMAND = cd /home/lesliefang/pytorch/pytorch && /home/lesliefang/pytorch/anaconda3/envs/pytorch_leslie/bin/python tools/setup_helpers/generate_code.py --declarations-path /home/lesliefang/pytorch/pytorch/build/aten/src/ATen/Declarations.yaml --nn-path aten/src
```
**/home/lesliefang/pytorch/pytorch/build/aten/src/ATen/Declarations.yaml 这个文件和torch/share/ATen/Declarations.yaml 是一个文件**，例如conv2d的函数就是申明在这个文件里面，通过查这个yaml文件，我们就可以知道Python函数名和C++函数名之间的映射关系

* tools/setup_helpers/generate_code.py:main ->
* tools/setup_helpers/generate_code.py:generate_code ->
* tools/autograd/gen_autograd.py:gen_autograd_python ->
这个函数首先通过去载入Declarations.yaml文件去解析所有的函数
```
aten_decls = load_aten_declarations(aten_path)
```
然后调用gen_py_torch_functions
* tools/autograd/gen_python_functions.py:gen_py_torch_functions
在这个函数里面去创建了python_torch_functions.cpp 文件
第一步：这个函数首先读取templates目录下面python_torch_functions.cpp的同名模板文件
第二步：然后通过get_py_torch_functions函数解析yaml文件里面的声明函数，创建py_torch_functions列表
第三步：调用create_python_bindings 函数进一步格式化yaml文件中读到的内容
第四步：调用wirte函数去创建torch/csrc/autograd/generated/python_torch_functions.cpp 文件，用yaml文件里面解析到的内容去替换template文件对应的位置

<!---more--->
### Pytorch的编译过程介绍
各个依赖，例如mkldnn的版本应该是通过git submodule的版本去控制的
通过生成wheel包的命令出发
```
python setup.py bdist_wheel
```
* setup.py:main
* setup.py:build_deps
* tools/build_pytorch_libs.py:build_caffe2
调用了cmake.generate 和 cmake.build函数
利用ninja去执行各个包目录下面的cmake命令
build/build.ninja 文件里面包含了所有要执行的cmake的命令
* setup.py:setup 函数生成wheel包
```
ext_modules=extensions,
cmdclass=cmdclass,
```
ext_modules 由 configure_extension_build函数生成，指定了依赖的C++动态链接库
cmdclass 指定了命令的映射
通过cmdclass会去执行build_ext::run 函数

### Debug 技巧
#### 打日志
Aten模块可以通过TORCH_WARN
定义在pytorch/c10/util/Exception.h/cpp文件里面

修改完代码，直接
```
python setup.py install
```
然后运行新的测试
