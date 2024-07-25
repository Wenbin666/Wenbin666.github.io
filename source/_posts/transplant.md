---
title: linux kernel 代码移植常见问题与解决
date: 2021-9-18 19:24:41
tags: C/C++
categories:  
  - 编程
---
# linux kernel 代码移植常见问题与解决

# 1. 通用Makefile写法
```Makefile
CROSS_COMPLE:="linux-gnu-"
ARCH:=arm

CC:=$(CROSS_COMPILE)gcc
LD:=$(CROSS_COMPILE)ld

KERNELDIR := /home/user/xxx/file/kernel
CURRENT_PATH := $(shell pwd)

obj-m := test_module.o
test_module-objs:= test.o #此处根据依赖填写

build: kernel_modules

kernel_modules:
    $(MAKE) -C $(KERNELDIR)  M=$(CURRENT_PATH) modules
clean:
    $(MAKE) -C $(KERNELDIR)  M=$(CURRENT_PATH) clean
```

# 2. Invalid module format
模块与内核版本不匹配的错误, 可以通过uname -a 查看系统版本，通过modinfo查看ko版本，确定内核与模块是否匹配

# 3. modprobe: Module ko not found
ko不在/lib/modules/kernel/下，可以先手动拷贝过去，再执行depmod 建立依赖关系，最后执行modprobe + 模块名称(无需带ko后缀)

# 4. module通过modprobe加载后不能正常运行且无log输出
模块可以正常加载，无报错信息，但是通过dmesg查看不到任何log，功能也不正常
重启后发现再次加载其实报错了：**loading out-of-tree module taints kernel**，这种错误只报一次，后续再次加载不再报错
提示信息中的taint是污染的意思，整个提示信息的意思是加载树外模块污染内核。当内核受到污染意味着内核处于社区不支持的状态，并且内核提供的某些功能可能会被禁用(pr_err/pr_info未打印等)；
产生taints可能的原因：
* 加载非GPL兼容的内核模块
* staging驱动程序的使用，它们是内核源代码的一部分，但尚未经过全面测试
* 使用内核源代码未包含的树外模块
* 强制加载不是为当前内核版本构建的模块
* 某些严重错误，例如machine check exceptions（MCE）和kernel oopses
更多细节可以查看：[loading out-of-tree module taints kernel](https://www.cnblogs.com/sky-heaven/p/13280047.html)

# 5. modpost：undefined
相关变量未找到，可以通过EXPORT_SYMBOL导出。
EXPORT_SYMBOL是Linux内核中一个常见API，其作用是讲一个symbol导出到内核空间，使得内核的所有代码都可以使用；

# 6. exported twice. Previous export was in vmlinux
二次编译，config中之前已经被配置为y了也就是已经链接到vmlinux里面了，再次尝试把它编译为模块就会出现这种情况；
可以在vmlinux.symvers中查看buildin已经导出的函数，Module.symvers中查看module导出的函数

# 7. MODULE_DEVICE_TABLE
对于USB、PCI等热插拔设备驱动，通常会创建一个MODULE_DEVICE_TABLE，以表明该驱动模块所支持的设备。
MODULE_DEVICE_TABLE的第一个参数是设备的类型,如usb或pci。后面一个参数是设备表,这个设备表的最后一个元素是空的,用于标识结束。
例:假如代码定义了USB_SKEL_VENDOR_ID是0xfff0,USB_SKEL_PRODUCT_ID是Oxfff0,当有一个设备接到集线器时,usb子系统就会检查这个设备的vendor ID和product ID,如果他们的值是0xfff0时,
那么子系统就会调用这个模块作为设备的驱动。

# 8. module_platform_driver
写驱动的时候用module_platform_driver这个宏可以少写很多代码。
宏的作用就是指定名称的平台设备驱动注册函数和平台设备驱动注销函数，并且在函数体内分别通过platform_driver_register()函数和platform_driver_unregister()函数注册和注销该平台设备驱动。

# 9. KBUILD_EXTRA_SYMBOLS
kbuild 需要完全了解所有符号，以避免发出有关未定义符号的警告。
构建外部模块时，会生成一个 Module.symvers 文件，其中包含所有未在内核中定义的导出符号。
可以使用 KBUILD_EXTRA_SYMBOLS 并提供 Module.symvers 文件的路径（如果它存在于模块目录以外的其他目录中）。
如在Makefile中指定
```Makefile
KBUILD_EXTRA_SYMBOLS += $(src)/xxx/Module.Symvers
```
# 10. MODULE_SOFTDEP 建立模块依赖关系
当module2使用了module1中的符号，用modinfo命令查看时，可以清楚的看到模块的依赖关系！
模块也会自动建立依赖关系，但是当module2没有使用module1中的任何符号，但在仍需要保证module2在module1后加载
要通过修改module2的代码来完成加载module1,将这一行代码添加到module2的代码中：
```C
MODULE_SOFTDEP("pre: module1");  //注意：冒号后面的空格不可以少
```
如果不想修改module2的代码，也可以通过修改module1的代码实现同样的功能
```C
MODULE_SOFTDEP("post: module2")
```
# 11. kernel下网络配置
```bash
ifconfig eth0 down
ifconfig eth0 hw ether xx:xx:xx:xx:xx:xx
ifconfig eth0 up
ifconfig eth0 xxx.xxx.xxx.xxx netmask xxx.xxx.xxx.xxx
route add default gw xxx.xxx.xxx.xxx
```

# 12. uboot下网络配置
```bash
setenv ipaddr xxx.xxx.xxx.xxx
setenv serverip xxx.xxx.xxx.xxx
setenv gatewayip xxx.xxx.xxx.xxx
setenv netmask xxx.xxx.xxx.xxx
setenv eth1addr xx:xx:xx:xx:xx:xx
saveenv
```
