---
title: MKLDNN
date: 2018-01-19 17:02:02
tags: 技术
---
## 介绍
MKLDNN是一个Intel开发的基于Intel芯片架构的开源库，提供用于深度学习的API.
[Intel 开源软件中心](https://github.com/01org)
[MKLDNN Github 主页](https://github.com/01org/mkl-dnn)
[教程主页](https://intel.github.io/mkl-dnn/)

## 编译安装mkldnn
```
1. Git clone mkldnn
2. source psxe
3. Cd mkldnn && mkdir build && cd build
4. CC=icc CXX=icpc cmake .. -DCMAKE_INSTALL_PREFIX=../mkldnn_install/
    ##debug版本编译加参数 -DCMAKE_BUILD_TYPE=Debug
5. CC=icc CXX=icpc make -j 66 && make install
```
make install之后：
如果第二步没有指定目录，会安装在/usr/local目录下面
Shared libraries (/usr/local/lib):
头文件：
Header files (/usr/local/include):
文档:
Documentation (/usr/local/share/doc/mkldnn):

##  编译链接代码：
[入门文档](https://software.intel.com/en-us/articles/intel-mkl-dnn-part-1-library-overview-and-installation)
这个文档内容有点问题：
具体编译的命令看github的readme：https://github.com/01org/mkl-dnn
首先配置环境变量
```
source psxe(应该只需要mkl模块)
export MKLDNNROOT=/home/automation/mkldnn/selfBuilt/mkl-dnn-0.12/mkldnn_install

G++ 编译
g++ -std=c++11 -I${MKLDNNROOT}/include -L${MKLDNNROOT}/lib simple_net.cpp -lmkldnn -o ./bin/simple_net_cpp
编译debug版本:
g++ -g -std=c++11 -I${MKLDNNROOT}/include -L${MKLDNNROOT}/lib simple_net.cpp -lmkldnn -o ./bin/simple_net_cpp

用icc编译的话：
icpc -std=c++11 -I${MKLDNNROOT}/include -L${MKLDNNROOT}/lib simple_net.cpp -lmkldnn -o ./bin_icc/simple_net_cpp
```
编译完的运行：
报错找不到mkldnn的动态链接库
添加动态链接库
```
export LD_LIBRARY_PATH=$LD_LIBRARY_PATH://home/automation/mkldnn/selfBuilt/mkl-dnn-0.12/mkldnn_install/lib

报错找不到mkl_rt：
source 一下mkl或者psxe

然后运行：
./bin_icc/simple_net_cpp
运行成功，不会报错
```
## Simple_Net Code API 代码的解释
https://software.intel.com/en-us/articles/intel-mkl-dnn-part-2-sample-code-build-and-walkthrough

## 调试simple_net.cpp代码
### 看一个primitive的例子：
#### step1：
line:117
convolution_forward(conv1_prim_desc, conv1_src_memory,
            conv1_weights_memory, user_bias_memory,
            conv1_dst_memory)
convolution_forward就是primitive的名字
#### step2：
看convolution_forward的构造函数调用了mkldnn_primitive_create
#### step3：
mkldnn_primitive_create的构造函数：
调用了primitive_desc->create_primitive
#### step4:
看primitive_desc->create_primitive
这个是个虚函数，需要看具体的primitive_desc子类的实现
用gdb去看看到底调用哪个实现函数
**debug**
```
gdb a.out
break primitive.cpp:35(mkldnn_primitive_create这个函数上)
单步调试到return primitive_desc->create_primitive(primitive, inputs, outputs);
看看调用了哪个kernel
bt
```
单步调试n，遇到return primitive_desc->create_primitive(primitive, inputs, outputs); s：step in这个实现
##### 看到create_primitive函数创建memory的时候调用了
```
#0  mkldnn::impl::cpu::cpu_memory_t::pd_t::create_primitive (this=0x634400, primitive=0x7ffffffea508, inputs=0x0,
    outputs=0x0) at /home/lesliefang/mkldnn/mkl-dnn-0.17.2/src/cpu/cpu_memory.hpp:45
```
##### reoder这个op对应了
```
/home/lesliefang/mkldnn/mkl-dnn-0.17.2/src/cpu/jit_uni_reorder.cpp:818
```
##### conv这个op
对应调用到了/home/lesliefang/mkldnn/mkl-dnn-0.17.2/src/cpu/
jit_avx512_common_convolution.hpp:46
```
jit_avx512_common_convolution_fwd_t的构造函数里面new jit_avx512_common_conv_fwd_kernel

jit_avx512_common_conv_fwd_kernel构造函数里面调用generate方法
generate方法就是jit去生成汇编，可以看到里面调用了xbyak的库函数
generate函数最后ker_ = getCode<decltype(ker_)>();把生成的代码放在ker_
这个类有个operator方法是真实运行的代码，函数里面会调用ker_(&args);
operator方法在这个类的execute_forward函数里面的parallel方法里面被调用
```
进一步看generate方法：
###### jcp变量
jcp是一个jit_conv_conf_t类型的成员变量
在jit_avx512_common_conv_fwd_kernel初始化成员列表里面做初始化jcp(ajcp)
ajcp是传进来的参数conf_.jcp_
conf_是jit_avx512_common_convolution_fwd_t的成员变量pd_t conf_;
在jit_avx512_common_convolution_fwd_t初始化成员函数列表里面构造conf_(\*pd)

generate函数开始从jcp里面读出输入输出变量的维度
计算会调用compute_loop函数，compute_loop根据jcp.ver类型选择调用compute_loop_vnni或者compute_loop_4fma

## 多线程计算的实现
需要链接openMP去实现
### step1:
全局搜
```
grep -rni "pragma omp parallel"
看到
src/common/mkldnn_thread_parallel_nd.hpp:47:#   pragma omp parallel num_threads(nthr)
src/common/mkldnn_thread_parallel_nd.hpp:154:#   pragma omp parallel
```
应该是封装在了这个parallel函数里面
```
#elif MKLDNN_THR == MKLDNN_THR_OMP
    if (nthr == 1) { f(0, 1); return; }
#   pragma omp parallel num_threads(nthr)
    f(mkldnn_get_thread_num(), mkldnn_get_num_threads());
```
一般用openMP的线程不用tbb的线程
### step2:
在src目录下面搜索parallel的函数调用
是在execute函数里面调用的，所以并行化是在stream提交了之后，调用kernel的execute的函数的时候执行的
继续以simple_net.cpp代码中的jit_avx512_common_convolution.hpp:46的jit_avx512_common_convolution_fwd_t作为例子
stream提交之后会调用jit_avx512_common_convolution_fwd_t中的execute函数，如果2D卷积的话调用execute_forward_2d函数
这个函数里面调用了parallel函数去做多线程运算
parallel函数的第二个参数就是每个线程要执行的函数，一般就是个lamba匿名函数
匿名函数里面通过jit_conv_ker_pipeline_ow_thr去调用kernel_->jit_ker执行运算

kernel_变量在_jit_avx512_common_convolution_fwd_t构造函数里面实现
kernel_ = new jit_avx512_common_conv_fwd_kernel(conf_.jcp_,
            \*conf_.attr());
jit_ker在jit_avx512_common_conv_fwd_kernel的构造函数里面通过jit_ker = (void (\*)(jit_conv_call_s \*))getCode();实现

## MKLDNN的primitive如何选择对应的kernel
在调用privimite->create_primitivate的时候会去遍历所有的kernel，在每个kernel的函数中都有一个init_conf的函数，在遍历所有的kernel的时候，会去看这个init_conf的函数是否满足条件，满足条件就意味着调用这个kernel，否则的话就看下个kernel

以_jit_avx512_common_convolution_fwd_t为例子
是在构造primitive desc的时候就选择好了使用哪个kernel
### step1:
在simple_net.cpp line:93
auto conv1_prim_desc
            = convolution_forward::primitive_desc(conv1_desc, cpu_engine);
创建primitive desc

### step2:
convolution_forward::primitive_desc构造函数中调用mkldnn::primitive_desc

### step3:
mkldnn::primitive_desc里面调用mkldnn_primitive_desc_iterator_create_v2

### step4:
mkldnn_primitive_desc_iterator_create_v2函数里面创建primitive_desc_iterator_t对象
这个函数比较难懂，记录下自己的理解
```
status_t mkldnn_primitive_desc_iterator_create_v2(
        primitive_desc_iterator_t **iterator, const_c_op_desc_t c_op_desc,
        const primitive_attr_t *attr, engine_t *engine,
        const primitive_desc_t *hint_fwd_pd) {
    const op_desc_t *op_desc = (const op_desc_t *)c_op_desc;

    auto it = new primitive_desc_iterator_t(engine, op_desc, attr, hint_fwd_pd);
    if (it == nullptr) return out_of_memory;

    ++(*it);
    if (*it == it->end()) {
        delete it;
        return unimplemented;
    }

    *iterator = it;
    return success;
}
```
* new primitive_desc_iterator_t的时候返回一个iterator，同时last_idx_遍历impl_list_所有op直到末尾，用于end()成员方法的调用。
impl_list_ 通过engine_->get_implementation_list() 在构造函数的初始化列表里面生成
engine_对应的cpu_engine 看get_implementation_list方法
是cpu_engine.cpp文件里面的函数，直接返回了cpu_impl_list变量
cpu_impl_list数组里面,每个元素都调用instance 宏创建了一个模板函数指针
istance宏其实是根据输入类型创建了mkldnn_primitive_desc::create这个模板函数指针（在primitive_desc.hpp文件里面）
在后面调用++it的时候，这段代码（auto s = impl_list_[idx_](&pd_, op_desc_, &attr_, engine_, hint_fwd_pd_);）其实就是调用了这个实例化mkldnn_primitive_desc::create这个模板函数指针
mkldnn_primitive_desc::create函数里面会去尝试创建对应的primitive description
mkldnn_primitive_desc::create方法调用失败返回unimplemented
调用成功，把创建的pd赋值给传入的参数，同时返回success
同时++it函数里面也会去判断是否success

Ps.
source insight里面搜索primitive_desc_iterator_t符号
using primitive_desc_iterator_t = mkldnn_primitive_desc_iterator;
impl_list_包含了所有kernel的实现
* ++(\*it); primitive_desc_iterator_t里面对++运算符进行了重载，在++重载的运算符中遍历尝试根据输入的类型去创建primitive desc（auto s = impl_list_[idx_](&pd_, op_desc_, &attr_, engine_, hint_fwd_pd_);），创建失败就尝试下一个primitive desc直到成功
* if (\*it == it->end()) 判断一下是不是创建成功退出的
* \*iterator = it; 创建成功的话，直接把it赋值给iterator返回
通过以上步骤就找到了对应要调用的primitive_desc
后面在创建primitive的时候调用primitive_desc->create_primitive，就是对应pd的create_primitive函数
通过下面对PD的分析可以知道这里调用到的就是pd里面DECLARE_COMMON_PD_T里面的create_primitive方法


### get_implementation_list在哪里实例化list里面的kernel
这个engine_是传进来构造的
看cpu_engine.cpp文件中get_implementation_list的实现line:331
```
const pd_create_f* cpu_engine_t::get_implementation_list() const {
    return cpu_impl_list;
}
```
这里的cpu_impl_list是个列表每个成员都是pd_create_f对象
#define INSTANCE(...) &primitive_desc_t::create<__VA_ARGS__::pd_t>
调用了primitive_desc_t::create<__VA_ARGS__::pd_t>创建了每个primitive desc的类型放在cpu_impl_list里面
#### using primitive_desc_t = mkldnn_primitive_desc;
primitive_desc_t就是mkldnn_primitive_desc
#### mkldnn_primitive_desc
mkldnn_primitive_desc有create方法，传进来的参数是__VA_ARGS__::pd_t
比如_jit_avx512_common_convolution_fwd_t就是这个op的pd_t成员
看mkldnn_primitive_desc有create方法，里面调用了_pd->init()，就是_jit_avx512_common_convolution_fwd_t的pd_t成员的init方法


## MKLDNN框架介绍
<!--more-->
### Caffe调用MKLDNN
可以我的另外一篇博客参考：Caffe中的生产者模式
1. caffe的layer_factory会去创建mkldnn的层
2. caffe在调用layer.forward函数的时候会调用到对应mkldnn的层的forward_cpu函数
3. 在这个函数中(initxxxpd)的方法会先去判断对应的pd(privimite descriptive)和privimite是否存在，如果不存在的话会去创建对应的pd
4. 在initxxxpd方法中会去调用reset的方法去创建pd(privimite descriptive)和privimite
5. reset方法的传入的参数就是new出来的一个新的mkldnn的primitive
**进入mkldnn**
6. 在new一个新的mkldnn的primitive的时候，可以看到create后缀或者init后缀的函数
7. 在这些create函数中最后会去调用一个privimite->create_primitivate的函数，这是个虚函数
8. 这个虚函数和具体的运算kernel之间如何调用实现的，**暂时不是很清楚**，应该有复杂的调用关系，但是核心的思想是：在调用privimite->create_primitivate的时候会去遍历所有的kernel，在每个kernel的函数中都有一个init_conf的函数，在遍历所有的kernel的时候，会去看这个init_conf的函数是否满足条件，满足条件就意味着调用这个kernel，否则的话就看下个kernel

## MKLDNN with VTune
https://github.com/intel/mkl-dnn/blob/master/doc/perf_profile.md#intelr-vtunetm-profiling

## kernel的名字
igemm_s8u8s32:blas
s8u8s32: s8表示输入的数据类型，u8表示权重的数据类型
术语：https://github.com/intel/mkl-dnn/tree/master/tests/benchdnn

## 运行benchdnn
https://github.com/intel/mkl-dnn/tree/master/tests/benchdnn
编译完mkldnn在build目录下面


## 环境变量
### vebose
export MKLDNN_VERBOSE=1
https://intel.github.io/mkl-dnn/perf_profile.html
### Dumping JIT-kernels
export MKLDNN_JIT_DUMP=1
https://intel.github.io/mkl-dnn/perf_profile.html

### VERBOSE如何工作
环境变量设置了VERBOSE=1的话mkldnn_verbose()->level的值就是1
看这个文件cpu_engine_t.cpp
在stream在submit的时候，如果设置了verbose就会打印出来
```
status_t cpu_engine_t::submit(primitive_t *p, event_t *e,
        event_vector &prerequisites) {
    /* FIXME: this should live in primitive execute function... */
    if (mkldnn_verbose()->level) {
        double ms = get_msec();
        p->execute(e);
        ms = get_msec() - ms;
        printf("mkldnn_verbose,exec,%s,%g\n", p->pd()->info(), ms);
        fflush(0);
    } else {
        p->execute(e);
    }
    if (msan_enabled)
        unpoison_outputs(p);
    return success;
}
```
p->pd()->info()就是这个primitive对应的pd的信息
p是mkldnn_primitive类型的
primitive.hpp文件定义了这个类的成员和函数
p->pd()
p->kind()

## 根据MKLDNN_VERBOSE输出内容查看对应调用到的primitive和kernel
### 设置verbose=1之后看到日志输入
关键字： jit:avx512_common

### mkldnn找到这两个文件
jit_avx512_common_convolution.hpp
jit_avx512_common_convolution.cpp

hpp文件里面订了了这个primitive的类，里面pd_t的结构体就是这个promitive的desciptive
调用pd->create_primitive的时候就调用了这个pd_t结构体里面的
```
DECLARE_COMMON_PD_T(
        JIT_IMPL_NAME_HELPER("jit:", avx512_common, ""),
        jit_avx512_common_convolution_fwd_t);
```
这个内联函数，里面创建了这个primitive的实例
看这个primitive的构造函数
```
jit_avx512_common_convolution_fwd_t(const pd_t *apd,
        const input_vector &inputs, const output_vector &outputs)
    : cpu_primitive_t(apd, inputs, outputs)
{
    kernel_ = new jit_avx512_common_conv_fwd_kernel(pd()->jcp_,
                *pd()->attr());
}
```
关键：
```
kernel_ = new jit_avx512_common_conv_fwd_kernel(pd()->jcp_,
            *pd()->attr());
```
### 这里引出了两个kernel的文件：
jit_avx512_common_conv_kernel.cpp
jit_avx512_common_conv_kernel.hpp

看jit_avx512_common_conv_fwd_kernel的构造函数
```
jit_avx512_common_conv_fwd_kernel(jit_conv_conf_t ajcp,
        const primitive_attr_t &attr)
    : jcp(ajcp), attr_(attr), eltwise_injector_(nullptr)
{
    if (jcp.with_eltwise)
        eltwise_injector_ = new jit_uni_eltwise_injector_f32<avx512_common>(
                this, jcp.eltwise);

    generate();
    jit_ker = (void (*)(jit_conv_call_s *))getCode();
}
```
generate();函数就是去生成汇编代码
jit_ker = (void (\*)(jit_conv_call_s \*))getCode();函数就是把汇编代码放到jit_ker这个kernel对象的成员变量里面（这个kernel对象又是这个primitive对象的成员变量）
getcode函数也会去dump bin文件(如果设置了环境变量:export MKLDNN_JIT_DUMP=1)

### stream submit之后发生了什么
stream->submit(pd)
之后调用了jit_avx512_common_convolution.hpp里面的exectue函数
exectue函数根据条件选择jit_avx512_common_convolution.cpp里面具体的exectue的实现
具体的exectue的实现里面，调用了parallel去做并行化的计算，每个线程里面调用kernel_->jit_ker（jit_ker按照上一节的解释就是对应的汇编代码）
