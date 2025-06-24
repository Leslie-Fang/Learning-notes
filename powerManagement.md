---
title: powerManagement
date: 2018-03-22 22:09:14
tags: 技术
---
## Power mode
设置cpupower模式为performance或者powersave
cpupower frequency-set -g performance(powersave)

## 修改kernel的启动参数
如何添加kernel启动的参数
vi /boot/efi/EFI/redhat/grub.cfg
找到对应的启动的kernel
找到这一行
linuxefi /vmlinuz-3.10.0-514.6.1.el7.x86_64 root=UUID=b3343e70-94d7-4506-88e2-025037204d23 ro crashkernel=auto rhgb quiet LANG=en_US.UTF-8

在这一行末尾添加参数，，以intel_pstate为例（https://www.kernel.org/doc/html/v4.14/admin-guide/kernel-parameters.html）
linuxefi /vmlinuz-3.10.0-514.6.1.el7.x86_64 root=UUID=b3343e70-94d7-4506-88e2-025037204d23 ro crashkernel=auto rhgb quiet LANG=en_US.UTF-8 intel_pstate=disable

reboot机器之后会生效

## Driver
有两种power manager的driver
intel_pstate 和 acpi-cpufreq

通过 cpupower frequency-info 可以查看到当前使用的驱动
通过在kernel设置intel_pstate=disable来使用acpi-cpufreq

intel_pstate会使用hardware P state control (HWP)去控制cpu的P state
acpi-cpufreq 不会使用hardware P state control (HWP)

使用intel_pstate时，fixfreq的脚本不起作用
只有set 0x774寄存器
set 一下fixfreq的脚本
rdmsr -a 0x774
rdmsr -a 0x198 读取0x198的寄存器看频率数据
cpupower monitor(freqency-info) 去看看
会看到数值之间有差距

使用acpi-cpufreq时，fixfreq的脚本起作用
set 一下fixfreq的脚本
rdmsr -a 0x774
rdmsr -a 0x198 读取0x198的寄存器看频率数据
cpupower monitor(freqency-info) 去看看

## 工作原理总结
官方介绍文档,第一个总体介绍cpu 频率的调节原则，第二个具体介绍了其中的一种pstate算法
https://www.kernel.org/doc/html/v4.19/admin-guide/pm/cpufreq.html
https://www.kernel.org/doc/html/v4.12/admin-guide/pm/intel_pstate.html
intel的owner rafael.j.wysocki@intel.com
看完官方文档再结合代码就明白了

原来linux是用acpi-drive去调节cpu 频率
新的Intel cpu用intel-pstate drive去调节CPU频率
后来intel又提供了Hardware P 去调节CPU频率
要完全使用intel-pstate，需要在bios里面把Hardware disable掉,或者启动kernel的时候添加参数intel_pstate=no_hwp
看kernel这个代码drivers\cpufreq\intel_pstate.c写了：HWP存在的时候，intel-pstate会被ignore掉

更底层的理解(可能不全面):操作系统的driver(acpi或者pstate)，根据目前cpu的负载(计算cpu在5种状态下的时间比例top命令可以看到)，计算得到的目标频率。请求ucode将这个目标频率写入某个pcode可以读的寄存器里面，pcode运行在硬件的pcu里面(pcode是一个单while循环，循环中间有checkpoint，每运行到一个checkpoint会去检查是否有更高优先级的事情要处理)，pcode读到这个目标频率之后生成一个rv,rv通过一系列的硬件序列化，检查阈值等操作得到最终的tv值，tv值就是真正的目标的运行频率。
pcu控制cpu的电压以及运行频率。

acpi-drive支持5种governor的调节模式:Performance,powersave,Userspace,Ondemand和Conservative
intel-pstate只支持两种governor:Performance和powersave，intel-pstate的powersave的策略不同于acpi-drive，更像是acpi-drive的Ondemand策略

利用kernel提供的cpupower工具可以查看当前kernel使用的driver以及governor类型：
```
root# cpupower frequency-info
analyzing CPU 0:
  driver: intel_pstate
  CPUs which run at the same hardware frequency: 0
  CPUs which need to have their frequency coordinated by software: 0
  maximum transition latency:  Cannot determine or is not supported.
  hardware limits: 1.20 GHz - 3.80 GHz
  available cpufreq governors: performance powersave
  current policy: frequency should be within 1.20 GHz and 3.80 GHz.
                  The governor "performance" may decide which speed to use
                  within this range.
  current CPU frequency: Unable to call hardware
  current CPU frequency: 1.20 GHz (asserted by call to kernel)
  boost state support:
    Supported: yes
    Active: yes
```
要修改driver，在boot选项中disable intel-pstate然后重启机器
要修改governor，可以hot-change，直接利用kernel工具,cpupower
```
cpupower frequency-set -g performance
```

通过查看kernel源码：tools\power\cpupower\lib\cpupower.c
```
static unsigned int sysfs_cpufreq_write_file(unsigned int cpu,
          const char *fname,
          const char *value, size_t len)
{
 char path[SYSFS_PATH_MAX];
 int fd;
 ssize_t numwrite;

 snprintf(path, sizeof(path), PATH_TO_CPU "cpu%u/cpufreq/%s",
    cpu, fname);

 fd = open(path, O_WRONLY);
 if (fd == -1)
  return 0;

 numwrite = write(fd, value, len);
 if (numwrite < 1) {
  close(fd);
  return 0;
 }

 close(fd);

 return (unsigned int) numwrite;
}
```
知道这个工具其实是往sysfs对应的文件写入performance
/sys/devices/system/cpu/cpu*/cpufreq/scaling_governor

sysfs将内核空间的一些数据结构的接口暴露到了用户空间

### 进一步看intel-pstate调节的代码
代码在driver/cpufreq模块里面，有三层：core提供代码执行的框架，governor提供计算目标频率的算法，driver提供操作硬件的接口
##### 整个初始化流程：

* 调用driver’s ->init() 函数：intel_pstate.c文件里面的intel_pstate_cpu_init函数
* 调用governor’s ->init() 和start函数: cpufreq.c文件cpufreq_init_governor以及cpufreq_start_governor函数
cpufreq_start_governor函数里面根据实例化的governor去调用start函数
如果是intel-pstate的话，governor安全被重写了成了intel_pstate.c的setpolicy函数

##### 正常工作的流程：
intel_pstate.c的setpolicy函数被kernel注册为
https://www.kernel.org/doc/html/v4.19/admin-guide/pm/cpufreq.html
文档里面有描述:
Instead, the driver’s ->setpolicy() callback is invoked to register per-CPU utilization update callbacks for each policy.
也就是说cpu的利用率更新的回调函数就是setpolicy函数，通过kernel去调度执行

##### 看驱动代码
drivers\cpufreq\intel_pstate.c
看cpufreq_driver这个结构体可以知道target和setpolicy必定要设置一个

driver的setpolicy函数会在drivers\cpufreq\cpufreq.c文件里面被调用到
cpufreq_set_policy函数里面会调用cpufreq_driver->setpolicy
这个就对应了具体driver的setpolicy函数


当driver是intel_pstate的时候
关键函数是setpolicy：intel_pstate_set_policy
首先调用intel_pstate_update_perf_limits 去获得最大以及最小频率值设置到cpudata这个结构体里面，cpudata这个结构体存了每个cpu实例的数据
intel_pstate_set_update_util_hook函数：这里hwp activate的话直接返回
否则调用cpufreq_add_update_util_hook函数，进一步调用intel_pstate_update_util函数
intel_pstate_update_util函数里面关键调用了intel_pstate_sample去读msr寄存器的值采集当前cpu的繁忙状态和intel_pstate_adjust_pstate函数去设置pstate，也就是写msr寄存器
hwp activate的话调用intel_pstate_hwp_set函数：
intel_pstate_hwp_set函数里面intel_pstate_get_epp函数去获得状态，intel_pstate_set_epb去设置pstate

pstate 设置成passive模式的时候，driver就是intel_cpufreq
运行时的关键函数是target：intel_cpufreq_target，这个函数的执行流程：
cpufreq_freq_transition_begin函数：等待直到被唤醒
intel_pstate_prepare_request函数：获得target_pstate
如果target_pstate和当前pstate不相等: 写msr寄存器wrmsrl_on_cpu(msr:0x00000199)
intel_cpufreq_trace函数:先调用intel_pstate_sample函数去采样获得当前的cpu的繁忙状态，再调用trace_pstate_sample函数
等待唤醒

drivers\devfreq目录下面有
各个governor对应的文件
governor_performance.c文件为例：
```
subsys_initcall(devfreq_performance_init);
```
subsys_initcall的作用就是将初始化函数注册到一个特定的段里面
内核初始化的时候调用do_initcalls决定是否调用这个段对应的初始化函数
这个文件里面定义一个devfreq_performance_func回调函数
```
static int devfreq_performance_func(struct devfreq *df,
        unsigned long *freq)
{
 /*
  * target callback should be able to get floor value as
  * said in devfreq.h
  */
 if (!df->max_freq)
  *freq = UINT_MAX;
 else
  *freq = df->max_freq;
 return 0;
}
```
直接把目标freq设置成最大值
所以performance governor很暴力，每一次都把目标频率设置成最大值，交给pcu去调节

### Intel-Pstate的turbo
turbo状态是 **不可持续** 的
每个cpu有个turbo threshold的频率值，通过读msr可以知道对应cpu的这个值：


高于这个值就是turbo状态，power不支持所有core同时进入turbo状态，也无法保证一个core在turbo状态下呆多久
低于这个值是非turbo状态，把频率fix在非turbo状态下的某个值，所有core都可以跑在这个值，不会低于这个值，但是某些core可以偶偶运行在高于这个值的频率上甚至turbo频率上面
