---
title: tensorflow编译系统
date: 2020-12-03 21:57:07
tags: 深度学习 Tensorflow
---
## 介绍
本文以一组tensorflow的常用编译命令为例，去介绍tensorflow的编译系统
```
export build_dir=$path_var #定义编译输出目录
yes "" | python configure.py
bazel --output_user_root=$build_dir build --config=mkl --config=opt --copt=-O3 -s //tensorflow/tools/pip_package:build_pip_package
bazel-bin/tensorflow/tools/pip_package/build_pip_package $build_dir
```

## Step1
命令：
```
yes "" | python configure.py
```
执行configure.py脚本，主要作用是把编译选项写入.tf_configure.bazelrc 文件
.tf_configure.bazelrc文件会被.bazelrc 文件import
```
try-import %workspace%/.tf_configure.bazelrc
```

我的理解.bazelrc 文件里面定义的就是hardcode的编译选项
.tf_configure.bazelrc文件通过configure.py暴露了一些可配置的选项给用户去配置

## Step2
命令：
```
bazel --output_user_root=$build_dir build --config=mkl --config=opt --copt=-O3 -s //tensorflow/tools/pip_package:build_pip_package
```
### 解析编译命令
* 根据--config的定义，解析.tf_configure.bazelrc 和 .bazelrc文件生成各个定义--define值

### 解析WORKSPACE文件
* WORKSPACE 里面加载了tensorflow/workspace.bzl 的tf_repositories函数
tf_repositories函数里面定义了各种外部依赖，一旦被调用到就会执行tf_http_archive 函数去下载并解压各种依赖库(包括**mkldnn**）
tf_http_archive 是一个repository_rule（详见下面术语的解释），函数定义在//third_party:repo.bzl 文件里面
**注意** 这里只是定义，不会立刻去下载，解压，只是定义了target，这个target一旦在某个地方呗调用到了，就会去执行下载，解压的操作

* WORKSPACE 里面加载了load("@bazel_tools//tools/build_defs/repo:http.bzl", "http_archive")
http_archive 函数也定义一些用于下载依赖的压缩包的target

### 调用target目标
#### build_pip_package 这个target
//tensorflow/tools/pip_package:build_pip_package  找到 //tensorflow/tools/pip_package目录下面BUILD文件，找到build_pip_package name
```
sh_binary(
   name = "build_pip_package",
   srcs = ["build_pip_package.sh"],
   data = COMMON_PIP_DEPS +
          select({
              "//tensorflow:windows": [
                  ":simple_console_for_windows",
              ],
              "//conditions:default": [
                  ":simple_console",
              ],
          }) + if_mkl_ml(["//third_party/mkl:intel_binary_blob"]),
)
```

这里最主要的依赖是COMMON_PIP_DEPS，从这里会延伸到各个需要build的target上面
一条我认为比较重要的编译依赖链路：
* //tensorflow/tools/pip_package:build_pip_package
* COMMON_PIP_DEPS
* //tensorflow/python:while_v2
* //tensorflow/python:framework_ops
* //tensorflow/python:platform
* //tensorflow/python:lib
* //tensorflow/python:pywrap_tensorflow
* //tensorflow/python:pywrap_tensorflow_internal
从pywrap_tensorflow_internal会调用到各个C/C++ 实现的依赖(core,kernel,op)上面

接上面看下怎么调用到下载编译mkldnn
* //tensorflow/c:c_api
* //tensorflow/c:c_api_internal
* //tensorflow/core:core_cpu
* //tensorflow/core/common_runtime:core_cpu
* mkl_deps()
如下面的解释，mkl_deps函数定义在third_party/mkl/build_defs.bzl 会去下载和编译MKKLDNN

<!--more-->
## Step3
命令：打包生成wheel包
```
bazel-bin/tensorflow/tools/pip_package/build_pip_package $build_dir
```
调用step2生成的target去打包生成wheel文件


## 各个文件的作用
* .bazelrc
bazel build的时候会从.bazelrc文件里面提取编译定义选项

* WORKSPACE
workspace文件，一个项目只有一个workspace文件

* BUILD
每个目录都有一个BUILD文件，里面定义了这个目录需要包含的所有target
比如这个target： //tensorflow/tools/pip_package:build_pip_package  就在//tensorflow/tools/pip_package/BUILD文件里面定义
BUILD文件可以去引用别的BUILD文件里面定义的target

* build_defs.bzl
一般定义了一些函数，可以被BUILD文件load去使用


## bazel build的各个关键字

### Build命令中的关键字
https://docs.bazel.build/versions/master/user-manual.html
* --config 用来使能bazelrc文件里面的选项 **-c的选项有可能是--config的缩写**
参考一部分.bazelrc 文件关于mkl的编译部分：
```
# Please note that MKL on MacOS or windows is still not supported.
# If you would like to use a local MKL instead of downloading, please set the
# environment variable "TF_MKL_ROOT" every time before build.
build:mkl --define=build_with_mkl=true --define=enable_mkl=true
build:mkl --define=tensorflow_mkldnn_contraction_kernel=0
build:mkl --define=build_with_openmp=true
build:mkl -c opt
```
build 后面接的都是默认生效的编译参数
build:mkl 后面接的编译参数只有当bazel build --config=mkl的时候mkl后面的编译参数才会起作用

* --copt： This option takes an argument which is to be passed to the compiler. 所以--copt后面传进来的都是gcc或者是icc的编译参数

* --strip 是否删除debug信息，never表示不删除debug信息
默认值是sometimes表示取决于其它选项
always表示会删除
https://docs.bazel.build/versions/master/user-manual.html#flag--strip

### build和bazel文件里面的关键字
* --define=var=val 定义一个变量var，它的值是val

* config_setting 根据bazel build传入的参数定义对应的变量
这个例子解释的很好
```
config_setting(
    name = "x86_mode",
    values = { "cpu": "x86" }
)
config_setting(
    name = "arm_mode",
    values = { "cpu": "arm" }
)
```
如果我们
bazel build target --cpu x86 那么x86_mode 这个变量就被定义了
bazel build target --cpu arm 那么arm_mode 这个变量就被定义了

* http_archive
定义在@bazel_tools//tools/build_defs/repo:http.bzl里面，用来下载压缩包
https://docs.bazel.build/versions/master/repo/http.html#http_archive

* load
去载入别的工具或者target
@： 表示从bazel默认的工具目录开始找
//： 表示从当前workspace的根目录开始找

* repository_rule
https://docs.bazel.build/versions/master/skylark/repository_rules.html
例子：
比如mkldnn的下载:
workspace.bzl 文件里面定义了
```
tf_http_archive(
    name = "mkl_dnn_v1",
    build_file = clean_dep("//third_party/mkl_dnn:mkldnn_v1.BUILD"),
    sha256 = "5369f7b2f0b52b40890da50c0632c3a5d1082d98325d0f2bff125d19d0dcaa1d",
    strip_prefix = "oneDNN-1.6.4",
    urls = [
    ],
)
```
third_party/mkl/build_defs.bzl 定义了
```
def mkl_deps():
    """Shorthand for select() to pull in the correct set of MKL library deps.

    Can pull in MKL-ML, MKL-DNN, both, or neither depending on config settings.

    Returns:
      a select evaluating to a list of library dependencies, suitable for
      inclusion in the deps attribute of rules.
    """
    return select({
        "@org_tensorflow//third_party/mkl:build_with_mkl": ["@mkl_dnn_v1//:mkl_dnn"],
        "@org_tensorflow//third_party/mkl:build_with_mkl_aarch64": ["@mkl_dnn_v1//:mkl_dnn_aarch64"],
        "//conditions:default": [],
    })
```

所以在某个地方调用到了mkl_deps()函数，此时如果--define=build_with_mkl=true 被定义了(通过.bazelrc文件)， ["@mkl_dnn_v1//:mkl_dnn"] 就会作为一个依赖被返回
作为依赖，["@mkl_dnn_v1//:mkl_dnn"] 就会被执行
这里的@mkl_dnn_v1 就是通过repository_rule(tf_http_archive(name = "mkl_dnn_v1"))定义的

## Example1：build_pip_package的target
这是个例子和解释是关于如何去阅读bazel的编译文件，**这个例子并不是在讲mkldnn的target是如何被编译到的**

//tensorflow/tools/pip_package:build_pip_package  找到 //tensorflow/tools/pip_package目录下面BUILD文件，找到build_pip_package name
```
sh_binary(
   name = "build_pip_package",
   srcs = ["build_pip_package.sh"],
   data = COMMON_PIP_DEPS +
          select({
              "//tensorflow:windows": [
                  ":simple_console_for_windows",
              ],
              "//conditions:default": [
                  ":simple_console",
              ],
          }) + if_mkl_ml(["//third_party/mkl:intel_binary_blob"]),
)
```
* src里面可以看到会去执行build_pip_package.sh 脚本
* data symbol的含义：https://docs.bazel.build/versions/master/be/common-definitions.html#common.data, data 后面跟着的就是这个target所有需要的文件
COMMON_PIP_DEPS 在同一个文件里面定义了一系列的依赖
* select：  {conditionA: valuesA, conditionB: valuesB, ...} 格式，满足条件A就执行值A :https://docs.bazel.build/versions/master/be/functions.html#select
config_setting根据bazel build传入的参数定义对应的变量
select根据config_setting定义的变量选择对应的表达式
select只能返回一个满足条件的选项，的时候如果多个条件都满足，选择一个条件最严苛的（子集）。如果几个满足条件的选项是并列的关系，bazel直接fail

* if_mkl_ml 是一个函数
加载//third_party/mkl/build_defs.bzl 文件，并且把这个文件里面的if_mkl，if_mkl_ml添加到当前BUILD文件中
在这里被加载
```
load("//third_party/mkl:build_defs.bzl", "if_mkl", "if_mkl_ml")
```

看 if_mkl_ml 函数的内容
```
def if_mkl_ml(if_true, if_false = []):
    """Shorthand for select()'ing on whether we're building with MKL-ML.

    Args:
      if_true: expression to evaluate if building with MKL-ML.
      if_false: expression to evaluate if building without MKL-ML
        (i.e. without MKL at all, or with MKL-DNN only).

    Returns:
      a select evaluating to either if_true or if_false as appropriate.
    """
    return select({
        "@org_tensorflow//third_party/mkl_dnn:build_with_mkl_opensource": if_false,
        "@org_tensorflow//third_party/mkl:build_with_mkl": if_true,
        "//conditions:default": if_false,
    })
```
看这个函数@org_tensorflow//third_party/mkl:build_with_mkl
```
config_setting(
    name = "build_with_mkl",
    define_values = {
        "build_with_mkl": "true",
    },
    visibility = ["//visibility:public"],
)
```
如果定义了build_with_mkl (build_with_mkl是在.bazelrc里面，如果--config=mkl就会被定义)，这个config_setting(name = "build_with_mkl")就会被定义，所以"@org_tensorflow//third_party/mkl:build_with_mkl"存在(满足条件)，if_true就会返回，if_true是传进来的参数在这里就对应了 //third_party/mkl:intel_binary_blob 这个依赖

#### intel_binary_blob 这个target
intel_binary_blob 定义在 //third_party/mkl/BUILD
```
cc_library(
    name = "intel_binary_blob",
    visibility = ["//visibility:public"],
    deps = select({
        "@org_tensorflow//tensorflow:linux_x86_64": [
            ":mkl_libs_linux",
        ],
        "@org_tensorflow//tensorflow:macos": [
            ":mkl_libs_darwin",
        ],
        "@org_tensorflow//tensorflow:windows": [
            ":mkl_libs_windows",
        ],
        "//conditions:default": [],
    }),
)
```
这里依赖于mkl_libs_linux
