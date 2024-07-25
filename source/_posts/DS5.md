---
title: DS5使用说明
date: 2024-07-15 19:30:00
tags: ARM-DS5
categories:  
  - Debug Tools
---

# DS5使用说明

# 1. 简介
DS-5是一款调试利器，可以用于调试裸板和linux系统，涵盖从启动代码到内核，再到应用程序的所有开发阶段。
文章结构：
![](image.png)

## 1.1 接口
基本接口：电源、网络、USB
AUX(Auxiliary): 辅助端口，用于未来对调试器本身的增强
USER IO: 连接任何用户I/O信号之前，先将DS5与目标设备共地
![](image-1.png)
PROBE: 调试器和目标设备的连接口
&emsp;**ARM JTAG 14**：比较老的ARM 标准接口，现在很少用
&emsp;**TI JTAG 14**： 连接TI系列的目标设备
&emsp;**ARM JTAG 20**：用的最多的ARM系列目标设备接口
&emsp;**MICTOR 38**： 最常用的trace 连接器接口（DS-5 Trace 功能实例 ）
&emsp;**MIPI 34**： 支持调试和trace信号的MIPI连接接口(支持两种电压信号VTREFA、VTREFB)
&emsp;**CORESIGHT 10/20**: 支持10路和20路CoreSight标准连接

## 2.2 按钮
正常复位：长按按钮，直到STATUS指示灯开始快速闪烁绿色时松开按钮
恢复模式：长按按钮10秒以上，STATUS指示灯快速闪烁红色时松开按钮，DSTREAM会重新启动并进入恢复模式，用于升级固件

## 2.3 指示灯
### 2.3.1 Debug功能
STATUS: 绿色代表正常，红色代表遇到错误需要复位调试器
FLASH: 在访问调试器内部FLASH的时候会显示为绿色
TARGET: 目标设备通过probe正确连接，并被识别到时显示为绿色
DEBUG:  调试数据正在传输时，DEBUG LED亮绿色

### 2.3.2 Trace功能
TRACING: 表示正在trace
TRIGGER: 表示触发器被检测到
DATA: 表示数据缓冲区中有有效数据\
FULL: 表示捕获了足够数量的trace数据并填满定义的缓冲区
TRC CLK: 检测到有效的trace 时钟显示为绿色，如果显示为红色表示trace时钟有问题

# 2. 功能和概念
## 2.1 主要功能
*	加载镜像和符号
*	运行镜像
*	设置断点(breakpoints)和观察点(watchpoints)
*	单步执行
*	访问变量和寄存器的值
*	查看内存的内容
*	查看堆栈情况
*	处理异常和Linux信号
*	调试裸板代码
*	调试多线程Linux应用程序
*	调试Linux内核和Linux内核模块
*	调试多核和多cluster系统(包含big.LITTLE系统)
*	调试实时操作系统（RTOS）
*	通过命令行调试
*	使用Streamline进行性能分析
*	GDB调试器命令
*	CoreSight和非CoreSight trace（Embedded Trace Macrocell Architecture Specification v3.0 and above）
*	支持小部分第三方CMM-style命令（cmm一种脚本语言）
## 2.2 调试器中的一些概念
*	AMP： (Asymmetric Multi-Processing)  非对称多处理系统有多个处理器, 处理器可能是不同的架构
*	SMP ：(Symmetric Multi-Processing) 对称多处理系统, 具有多个具有相同架构的处理器
*	Bare-metal：不带操作系统的裸机程序
*	Configuration database：配置数据库，用于储存DS-5可以连接的处理器、设备和板块信息（.xml files, python scripts, .rvc files, .rcf files, .sdf files等）；也可以通过platform configuration editor 工具生成自己的数据库文件。
*	Debug target：程序运行的环境(可以是硬件、模拟硬件的软件或硬件模拟器 )
    + 硬件：如开发板
    + 模拟硬件的软件：如FVP(Fixed Vittual Platform)是Arm的一种软件模型，它提供与真实硬件等效的功能行为
    + 硬件模拟器：产品开发的早期，如果没有可用的硬件，可以使用仿真器或软件目标来模拟硬件行为，如FPGA等

# 3. ARM Development studio
## 3.1 基本安装
根据工作系统功，正常下载相关安装包安装即可，此处不进行赘述
安装后界面如下：
![](image-2.png)

## 3.2 常用试图
### 3.2.1 Project Explorer

![](image-3.png)

* 说明：项目浏览器视图
* 主要功能：显示现有的数据库、连接文件夹等
* 常用功能：查看相关文件内容、Build Platform
### 3.2.2 ASM、C\C++ editer
![](image-4.png)
* 说明：编辑视图
* 主要功能：用于编辑或者显示Asm和C、C++等源文件，光标放到asm指令上会显示指令说明
* 常用功能：打断点或者取消断点、设置或取消trace起始点、设置或取消trace结束点、设置trace 触发点
### 3.2.3 Debug Control
![](image-5.png)
* 说明：调试控制视图
* 主要功能：显示正在运行的内核、线程或用户空间进程列表、显示运行等级：EL0、EL1、EL2、EL3、显示处理器模式：USR FIQ IRQ SVC ABT UND SYS
* 常用功能：与目标设备建立或断开连接、加载image或symbol、运行或停止程序、单步执行( **Debug from main**)、设置或修改源代码路径、reset目标设备、Debug configuration、暂停时显示调用栈
### 3.2.5 Diassembly
![](image-6.png)
* 说明：反汇编试图
* 主要功能：可以查看源代码的反汇编代码、显示地址空间信息：EL3、EL2、N等
* 常用功能：设置Next Instruction会在单步时自动更新、可以输入地址或函数名如main、$PC+256、History 显示曾经输入过的内容、设置或取消断点
### 3.2.6 Variables
![](image-7.png)
* 说明：变量视图
* 主要功能：显示局部变量、静态变量和全局变量的值
* 常用功能：手动增加用户关注变量、改变变量的显示（十进制、十六进制）、双击变量改变变量的值（方便）
### 3.2.7 Register
![](image-8.png)
* 说明：寄存器视图
* 主要功能：可以查看目标设备上寄存器的值
* 常用功能：双击改变寄存器的值、筛选关注的寄存器、查看寄存器(或某一位)的解释
### 3.2.8 Memory
![](image-9.png)
* 说明：内存视图
* 主要功能：可以修改视图中内存的显示方式
* 常用功能：改变显示方式，16进制，字符等、通过&查看变量在memory中的内容，通过sizeof方式获取长度、设置自动刷新
### 3.2.9 Command
![](image-10.png)
* 主要功能：可以输入各种命令并进行命令行调试
* 常用功能：输入命令、运行命令脚本、清除命令打印、通过Alt+/ 补全查看命令
### 3.2.10 stack view
![](image-11.png)
* 说明：栈视图
* 主要功能：可以根据选定的执行上下文查看内核、线程或进程的堆栈信息
* 常用功能：当目标设备stop的时候查看stack信息、查看调用栈情况和相关变量的值、更改查看栈条目数量
### 3.2.11 Breakpoints、watchpoint
![](image-12.png)
* 说明：断点视图
* 主要功能：显示程序中设置的：breakpoints、watchpoints和tracepoints、删除所有断点等
* 常用功能：禁用、启用或删除breakpoints、watchpoints和tracepoints
### 3.2.12 Debug hardware firmware installer
* 说明：调试器硬件更新视图
* 主要功能：DS-5的firmware有时会损坏，灯不亮或者一直显示红灯，可以通过此视图更新调试器的固件
* 常用功能：
  + 长按按钮10秒以上，STATUS指示灯快速闪烁红色时松开按钮，进入恢复模式，Debug Hardware可以识别到USB
  + Firmware update File一般会自动识别，如果识别不到，选择安装路径sw/debughw/firmware/下固件文件
  + 点击install并等待安装完成
# 4. 调试命令
DS-5的命令以group进行分类：
可以分为 breakpoints、cache、data、display、flash、info、log、memory、mmu、mpu、os、registers、running、scripts、set、show、stack、support共18组
**注意**：DS-5命令非常多，可以通过help + group/ command 命令查看命令的具体帮助内容
<img src="image-13.png" alt="" style="zoom:67%;" />

## 4.1 命令通用性规则
### 4.1.1 基本语法
命令样式:
command [argument] [/flag]...
基本例子:
<u>i</u>nfo <u>s</u>tack [n | -n] [full]

*	方括号[…]包含可选参数
*	 大括号{…}包含所需参数
*	垂直管道|表示必须从中选择一个
*	可以重复的参数后跟省略号（…）
*	下划线代表缩写，比如info stack可以缩写为is
*   full是否显示局部变量的值
### 4.1.2 路径相关特殊字符和环境变量
*	Home directory: ~
*	环境变量：（三种表示方式）
    + %LOG_DIRECTORY%
    + ${LOG_DIRECTORY}
    + $LOG_DIRECTORY
*	 反斜杠（\）或正斜杠（/）作为目录分隔符
### 4.1.4 调试器变量
| 变量        | 描述                                            |
|-----------|-----------------------------------------------|
| $         | print 命令等命令执行后的最后一次结果或其他命令的ID，储存在$中           |
| $$        | print 命令打印的倒数第二次结果或其他命令的ID，储存在$$中             |
| $n        | breakpoint、watchpoint、start、memory等命令的倒数三次及以上 |
| $cdir     | 当前工作目录                                        |
| $entryp   | 当前镜像的入口点                                      |
| $idir     | 当前镜像目录                                        |
| $sdir     | 当前脚本目录                                        |
| $datetime | 字符串格式的当前日期和时间                                 |
| $pid      | 当前操作系统进程ID                                    |
| $thread   |  多线程应用程序的当前线程ID                               |
| $core     | 当前处理器ID用于对称多处理（SMP）系统                         |
| $vmid     | 当前虚拟机ID（VMID），用于支持管理程序/虚拟机调试的系统               |
### 4.1.5 通配符的用法
符号\* ：指定0个或多个字符, ？ : 指定一个字符, \\ : 转义字符前缀符号
## 4. 命令File
功能：与文件交互相关的命令，主要用于控制image在目标上的加载和卸载，还有添加调试信息如symblo等
### 4.1.1 add-symbol-file
作用：将调试信息加载到调试器中(加载之后单步执行才会看到源码)
```C
// Syntax
add-symbol-file filename [offset] [-s section address]...
```
*	filename 指定镜像、共享库或操作系统（OS）模块。
*	offset 指定添加到image中所有地址的偏移量。如果未指定偏移量，则默为以下值：
    +	如果是image，则值为0
    +	如果是共享库，是库的加载地址
*	s 对于可重定位对象，这指定部分重定位的地址。
示例：
```C
add-symbol-file E:\task\vmlinux              // 调试 linux kernel
add-symbol-file D:\ARM64\Build\Firmware.elf  // 调试 binary 二进制
reload-symbol-file V:\mcu\Build\app_ram.elf // reload 命令用于再次load symbol
// 注意⚠️：需要保证运行镜像和symbol文件一致性！
```
### 4.1.2 restore
作用：从文件中读取数据并将其写入内存
```C
// Syntax
restore filename [binary] [offset [start_address [end_address|+size]]]
```
*	filename 指定文件
*	binary 指定二进制格式（文件格式仅对二进制文件是必需的，调试器会自动识别所有其他文件）
*	offset 指定在写入内存之前添加到镜像中所有地址的偏移量
*	start_address 指定可写入的最小地址，如果没有给出start_address，则默认为地址零
*	end_address指定可写入的最大地址，如果没有给出end_address，则默认为地址空间的末尾
*	size 指定区域的大小
```C
restore  "/test/app.bin" binary 0x0D800000
// Restore content of binary file  app.bin starting at 0xD800000
```
**后续其他命令具体细节可以查看官方手册说明，此处只列出简要说明！**
**append**: 从内存或表达式的结果中读取数据并将其附加到现有文件中

```shell
append memory myFile.bin 0x8000 0x8FFF
# Append content of memory 0x8000-0x8FFF to binary file myFile.bin
```
**directory**: 定义用于搜索source code的其他目录
```shell
directory "\usr\source"
# Add directory to search list
directory
# Reset to the default directories
```
**discard-symbol-file**: 丢弃与特定文件相关的调试信息
```shell
discard-symbol-file myFile.axf
# Discard symbols relating to myFile.axf
```
**dump**: 从内存或表达式的结果中读取数据并将其写入文件
```shell
dump memory myFile.bin 0x8000 0x8FFF
# Write content of memory 0x8000-0x8FFF to binary file myFile.bin
```
**file, symbol-file**: 将镜像中的调试信息加载到调试器中，并记录入口点地址以供运行和启动命令将来使用
```shell
file "myFile.axf"
# Load debug information on demand
file "images\myFile.axf"
# Load debug information on demand.
file
# Discard debug information.
```
**info files, info target**:显示有关加载的镜像和符号的信息
```shell
info files
# Display information for loaded image and symbols
```
**info sources**:显示正在调试的当前映像中使用的源文件的名称
```shell
info sources
# Display the names of source files
```
**loadfile**:将调试信息加载到调试器中，将镜像加载到目标上，并记录入口地址以供运行和启动命令将来使用
```shell
loadfile "myFile.axf"
# Load image and debug information when required
```
**pwd**:显示当前工作目录
```shell
pwd
# Display current working directory
```
**show directories**：显示调试器在搜索源文件时使用的搜索路径替换规则
```shell
show substitute-path
# Display all substitution rules.
```
### 4.1.3 Execution control
控制程序执行
**run**: 开始运行目标设备
```shell
run  # Start running the device
```
**interrupt, stop**:中断目标并在应用程序正在运行时停止应用程序
```shell
interrupt  # Interrupt application.
stop # Interrupt application.
```
**continue**: 继续运行目标
```shell
continue  # Continue running target
continue 5  # Continue running target, ignoring current breakpoint 5 times
```
**start**: 设置临时断点，调用调试器run命令，然后在命中临时断点时将其删除。默认情况下，临时断点设置在全局函数main()的地址
```shell
start   # Start running the target to the temporary breakpoint.
```
**next**:在源代码级别单步执行
```shell
next # Execute one source line
next 5 # Execute five source lines
```
**nexti**:指令级别单步执行
```shell
nexti # Execute one instruction
nexti 5 # Execute five instructions
```
### 4.1.4 Set相关命令
控制默认调试器设置的DS-5调试器命令
**set variable**：计算表达式并将结果分配给变量、寄存器或内存
```shell
set variable $PC=0x8000
set var $PC=0x8000
set variable $CPSR.N=0  # Clear N bit
set $PC=0x8000
```
**set listsize**：修改list命令显示的默认源行数
```shell
set listsize 20 # Set listing size for list command
```
**set endian**：指定调试器使用的字节顺序
```shell
set endian little # Debug using little endian
```
### 4.1.5 Data相关命令
显示源代码、表达式、变量、函数、类、内存和其他数据
**list**：显示当前或指定位置周围的源代码行
```shell
list main
list dhry_1.c:10,23
```
**info address**：显示symbol的位置
```shell
info address mySymbol
# Display location of symbol
```
**info locals**：显示当前堆栈帧的所有局部变量
```shell
info locals
```
### 4.1.6 Breakpoints 和 Watchpoints
**break**：在特定位置设置执行断点
```shell
break main # Set breakpoint at address of main()
break +1 # Set breakpoint at address of next source line
```
**clear**：删除特定位置的断点
```shell
clear *0x8000 # Clear breakpoint at address 0x8000
clear main # Clear breakpoint at address of main()
```
**disable breakpoints**：禁用一个或多个断点或观察点
```shell
disable breakpoints 1 # Disable breakpoint number 1
disable breakpoints 1 2 # Disable breakpoints number 1 and 2
disable breakpoints # Disable all breakpoints and watchpoints
```
**enable breakpoints**：按数字启用一个或多个断点或观察点
```shell
enable breakpoints 1 # Enable breakpoint number 1
enable breakpoints 1 2 # Enable breakpoints number 1 and 2
enable breakpoints # Enable all breakpoints and watchpoints
```
**ignore**：为断点或观察点条件设置忽略
```shell
ignore 2 3 # Ignore breakpoint 2 for 3 hits
```
### 4.1.7 Display and log
**echo**：仅显示文本字符串
```shell
echo " initializing..." # Display: " initializing..."
echo 4+4 # Display: 4+4
```
**x**：显示特定地址的内存内容
```shell
x 0x8000 # Display memory at address 0x8000
```
**log config**：指定从调试器输出运行时消息的日志配置类型
```shell
log config debug # Display all debug messages
```
### 4.1.8 Register
**info all-registers**： 显示当前堆栈帧的分组寄存
```shell
info all-registers # Display info for all registers
```
### 4.1.9 Cache
**cache flush**：刷新当前CPU的缓存
```shell
cache flush
```
**cache list**：列出当前核心可用的缓存和相关信息。输出是实现定义的
```shell
cache list  # Lists the available caches and views.
An example output is:
L1D:
  L1 data cache, size=32k, views: [tags, tlb]  ...
L1I:
  L1 instruction cache, size=2k, views: [tags, tlb]
```
### 4.1.10 Memory
**memory**：定义内存区域并指定其属性和大小
```shell
memory 0x1000 0x2FFF cache
# specify RW region 0x1000-0x2FFF (cache)
```
**enable memory**：启用一个或多个用户定义的内存区域
```shell
enable memory 1 # Enable region number 1
enable memory 1 2 # Enable regions number 1 and 2
```
**disable memory**：禁用一个或多个用户定义的内存区域
```shell
disable memory 1 # Disable region number 1
```
**memory set**：写入内存
```shell
memory set 0x8000 0 "Hello" # Writes a string to memory
```
### 4.1.11 Show
**show backtrace**：显示与info stack命令一起使用的行为设置
```shell
show backtrace limit # Display current call stack limit
```
**show breakpoint**：显示断点和监视点行为设置
```shell
show breakpoint skipmode
```
**show endian**：显示调试器正在使用的字节顺序设置
```shell
show endian # Display byte order setting.
```
### 4.1.12 Flash相关
**flash load**：将镜像中的部分加载到一个或多个闪存设备中
```shell
flash load "foo.axf" # loads the file to flash
```
**info flash**：显示有关当前target上的闪存设备的信息
```shell
info flash
```
### 4.1.13 Operating System (OS)
**sharedlibrary**：从共享库加载符号
```shell
sharedlibrary # Load symbols from all shared libraries.
sharedlibrary m* # Load symbols matching path starting with m
```
**info os**：显示操作系统（OS）支持的当前状态。如果启用OS支持，还列出所有可用的OS数据表
**info os-modules**：显示支持此功能的连接的可加载内核模块列表
**info processes**：显示有关用户空间进程的信息
**info threads**：显示有关可用线程的信息

### 4.1.14 Call Stack
**down**：将当前帧指针从调用堆栈向下移动并显示到底部帧。
**up**：将当前帧指针沿调用堆栈向上移动并显示到顶部帧
**info frame**：在选定位置显示堆栈帧信息

### 4.1.15 Support
**help**：显示根据特定调试任务列出的特定命令或一组命令的帮助信息
**quit, exit**：退出调试器会话
**define**：从现有命令派生新的用户定义命令
```shell
define add-args
   print $arg0+$arg1+$arg2
end
```
**trace list**：列出跟踪捕获设备和跟踪源
```shell
trace list # List all of the trace capture devices and trace sources
```
### 4.1.16 MMU相关
**mmu list tables**：列出可用的转换表及其关联参数
```shell
mmu list tables
Available translation tables:
PL1S_S1_TTBR0
   parameters: S_TTBCR, S_TTBR0, S_SCTLR PL1S_S1_TTBR1
   parameters: S_TTBCR, S_TTBR1, S_SCTLR  PL1N_S1_TTBR0
   parameters: N_TTBCR, N_TTBR0, N_SCTLR  PL1N_S1_TTBR1
   parameters: N_TTBCR, N_TTBR1, N_SCTLR
```
**mmu list translations**：列出可用的转换类型及其相关参数
**mmu translate**：执行虚拟地址和物理地址之间的转换
```shell
mmu translate 0x00008000 PL1S_S1 S_TTBR1=0x80000404A SP:0x80F15000
mmu translate SP:0x80F15000 Address SP:0x80F15000 maps to 0x00008000 0x80F15000
```
**mmu memory-map**：打印内存映射
# 6. 远程使用
## 6.1 配置公网IP与远程连接
Window --> Show view -->Debug Hardware Configure IP
<img src="image-14.png" alt="" style="zoom:67%;" />
通过这种方式为DS5配置一个公网IP，然后通过网络的方式可以访问DS5(这种配置的前提是本身已经通过USB与DS5建立连接)；
<img src="image-15.png" alt="" style="zoom: 50%;" />

## 6.1.2 远程查看状态与重启
```shell
http://xxx.xxx.xxx.xxx
```
 <img src="image-16.png" alt="" style="zoom:50%;" />

# 7. uboot与kernel调试
uboot会有relocate过程，如果调试relocate之前的过程可以直接添加符号表，如果调试之后的过程，需要在添加符号表时，加上relocate后的地址；
```shell
add-symbol-file file_name + relocate_after_addr
```
kernel 可以直接添加vmlinux进行调试
