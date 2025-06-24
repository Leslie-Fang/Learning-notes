---
title: 编译Ubuntu内核
date: 2017-10-30 19:30:28
tags: 技术
---
# 介绍
根据使用需要，有时候关闭一些内核选项，然而在OS没有提供对应工具的情况下，就需要重新[编译linux的kernal](http://www.cnblogs.com/xiaocen/p/3717993.html)

# 查看内核版本
```
# 查看内核版本
uname -r
# 查看OS版本
lsb_release -a
```

# 下载源码
apt-get install linux-source
下载内核
默认下载到 /usr/src 目录

# 解压缩包
```
 tar xf linux-.*.tar.xz -C /usr/src/
 ln -sv linux-.* linux # 有需要的话需要创建一个链接
```

# 配置内核选项并编译
## 配置编译选项
配置内核的方法的编译选项的方法有好多种，下面的每一种make就对应了一种方法，只需要从里面选一种就可以了，最常见的就是make menuconfig, 但是需要安装cc和ncurses-devel这两个包
```
make config：遍历选择所要编译的内核特性
make allyesconfig：配置所有可编译的内核特性
make allnoconfig：并不是所有的都不编译，而是能选的都回答为NO、只有必须的都选择为yes。
make menuconfig：这种就是打开一个文件窗口选择菜单，这个命令需要打开的窗口大于80字符的宽度，打开后就可以在里面选择要编译的项了
下面两个是可以用鼠标点选择的、比较方便哦：
make kconfig(KDE桌面环境下，并且安装了qt开发环境)
make gconfig(Gnome桌面环境，并且安装gtk开发环境)
menuconfig：使用这个命令的话、如果是新安装的系统就要安装gcc和ncurses-devel这两个包才可以打开、然后再里面选择就可以了、通这个方法也是用得比较多的：
```
**另外一种常见的方法是基于当前OS的编译配置选项去修改，复制当前系统上的/boot/config-版本-平台编译选项文件，将这个文件复制到/usr/src/linux/.config覆盖./config这个文件，然后make oldconfig去配置：这种方法下面，相同的选项就用.config里面老的配置，新的选项会去提示我们怎么配置**
```
c /boot/config .config
yes "" | make oldconfig#(使用.config里面的配置，.config里面没有配置的地方使用默认的选项)
```
## 编译
用screen开一个子窗口，在子窗口中编译
```
make -j144 all 2>&1 | tee make_record.txt
```
**centos上面报错找不到openssl的某个文件**
```
## 安装一下这个package
yum install openssl-devel
```
<!--more-->
**这一步不确定是否需要,目前感觉不一定需要。目前看下来，是编译的镜像要在别的机器上用的时候才需要，如何是在编译的本地机器上使用的话，应该是不需要**
make 之后 在 make install之前还[需要](https://bbs.archlinux.org/viewtopic.php?id=79851)
make bzImage一下，不然会报错
*** Missing file: arch/x86/boot/bzImage

# 编译后的安装
```
make modules_install #这一步是需要的
   这步完了之后你可以查看一下/lib/modules/目录下就会生成一个以版本号命名的一个文件模块了
make install
      安装完之后会在/boot/目录下生成一个内核文件vmlinuz-3.13.2、还有几个跟你当前编译的版本一样的文件、可以ls去看一下：
ls /boot/
```
# 配置启动项
编译好了一个新内核了之后可以到[grub.conf配置文件](http://www.nenew.net/ubuntu-grub-cfg.html)时看一下：
  # vim /boot/grub/grub.conf
[参考文档2](http://www.jinbuguo.com/linux/grub.cfg.html)
## How GRUB2 selects which kernel to boot from
https://www.thegeekdiary.com/centos-rhel-7-change-default-kernel-boot-with-old-kernel/
By default, the value for the directive GRUB_DEFAULT in the /etc/default/grub file is “saved”.
```
# cat /etc/default/grub
GRUB_TIMEOUT=5
GRUB_DEFAULT=saved
GRUB_DISABLE_SUBMENU=true
GRUB_TERMINAL_OUTPUT="console"
GRUB_CMDLINE_LINUX="nomodeset crashkernel=auto rd.lvm.lv=vg_os/lv_root rd.lvm.lv=vg_os/lv_swap rhgb quiet"
GRUB_DISABLE_RECOVERY="true"
```
This instructs GRUB 2 to load the kernel specified by the saved_entry directive in the GRUB 2 environment file, located at /boot/grub2/grubenv.


**真正起作用的是/boot/efi/EFI/centos下面的grubenv和grub.cfg**
而不是/boot/grub2/grub下面grubenv和grub.cfg
因为我们用的是UEFI的BIOS去启动的

重启系统，在重启界面上可以看到各个可以启动的内核的信息，选择我们刚刚编译好的这个内核
如果在第一层没有看到内核的选项，进入到advance的里面一级选项，里面会看到编译好的内核的启动选项

# 删除不需要的kernal
直接在/boot下面把对应的文件删除，一个kernal应该有三个文件
然后在/boot/grub/grub.cfg文件里面把对应的启动项给删除了
修改grub.cfg是只读属性
chmod 644 grub.cfg
改完保存之后
chmod 444 grub.cfg 重新改成只读的属性
启动之后看到旧的选项被删除了

# 添加debug log
利用kernel提供的printk函数添加log
https://www.kernel.org/doc/html/latest/core-api/printk-basics.html
https://www.cnblogs.com/aaronLinux/p/6843131.html
查看是否打印对应的内容
```
dmesg | grep log_content
```
