---
title: eigen
date: 2019-08-30 06:57:44
tags: 技术
---
## 介绍
数值计算库，在tensorflow中作为默认的数学库

## 安装
直接下载即可，不需要安装
全部都是头文件

## 测试安装
编写 test.cc
```
#include <iostream>
#include <Eigen/Dense>
using Eigen::MatrixXd;
int main()
{
  MatrixXd m(2,2);
  m(0,0) = 3;
  m(1,0) = 2.5;
  m(0,1) = -1;
  m(1,1) = m(1,0) + m(0,1);
  std::cout << m << std::endl;
}
```
编译：
g++ -I ./eigen/eigen-eigen-323c052e1731 test.cc -o test

## 计算对数值
```
#include <iostream>
#include <Eigen/Core>
#include "Eigen/src/Core/functors/UnaryFunctors.h"
#include <iomanip>
using namespace Eigen;
using namespace std;
int main()
{
  double res = Eigen::internal::scalar_log_op<double>()(280.41303540865516);
  std::cout << res << std::endl;
}
```

## 修改数组
使用unaryExpr的方法修改数组中的每个元素
```
#include <iostream>
#include <Eigen/Core>
#include "Eigen/src/Core/functors/UnaryFunctors.h"
#include <iomanip>
using namespace Eigen;
using namespace std;
int main()
{
  Eigen::Matrix<float, 100, 1> tmp = Eigen::Matrix<float, 100, 1>::Constant(100, 1, 280.41303540865516);
cout << setprecision(8) << tmp << endl << "becomes: " << endl << tmp.unaryExpr(internal::scalar_log_op<float>()) << endl;
}
```

tensorflow的Log等去对数的函数就调用了[unaryExpr方法](http://eigen.tuxfamily.org/dox/classEigen_1_1ArrayBase.html#title85)去计算数组的每个元素的对数
```
#./tensorflow/core/kernels/cwise_ops_common.h:541:
Assign(d, out, in.unaryExpr(typename Functor::func()));
```
