---
title: ARM64 Exception异常处理
date: 2023-10-23 20:53:10
tags: Execption
categories:  
  - CPU架构
---
# ARM64 Exception异常处理

# 1. 异常的概念
对于ARM64而言，exception是指cpu的某些异常状态或系统的一些事件（可能来自外部，也可能来自内部）导致cpu去执行一些预先设定的，具有更高执行权利的软件（exception handler）的过程。
当异常发生时，处理流程一般如下：
* 当前上下文保存：系统保存当前执行上下文（如寄存器、程序计数器等）
* 异常类型识别：确定引发异常的具体原因，可能需要查阅异常向量表
* 异常处理程序执行：根据异常类型，跳转到相应的异常处理程序
* 恢复执行上下文：在处理完异常后，将之前保存的上下文恢复，并继续执行程序，从异常发生的地方或者从某个安全点恢复
异常处理是操作系统和硬件之间的一个重要机制，用于保证系统的稳定性和安全性。

# 2.异常等级
![](image.png)
一旦异常发生，系统将切换到具有**更高执行权利**的状态，cpu需要切换异常等级(exception level)。
ARM64最大支持EL0～EL3四个等级，EL0的异常优先级最低，EL3的异常优先级最高, 根据Security状态不同进行了如下细分：
Normal World:
**EL0**: 运行用户空间的应用程序
**EL1**：运行guset os 如linux
**EL2**：Hypervisor虚拟化管理程序
Secure World:
**EL0**:Trusted service，可以理解为可信环境下的应用程序
**EL1**:Trusted Os， 如optee
**EL3**:用于运行Secure world的管理程序(BL31)
注意：在操作系统里,处理器运行模式通常分成两种:一种是特权梯式,另一种是非特权模式。操作系统内核运行在特权模式,访问系统的所有资源;而应用程序:运行在非特权模式,它不能访问系统的某些资源,因为它权限不够。上文提到的EL0就是非特权模式，不能进行异常处理。

# 3.异常的种类
# 3.1 中断(Interrupt)
在ARM64处理器中,中断请求分成普通中断请求(Interrupt Request, IRQ)和快速中断请求(Fast Interrupt Request,FIQ)两种, FIQ的优先级要高于IRQ。
在芯片内部, 分别有连接到处理器内部的IRQ和FIQ两根中断线。通常系统级芯片内部会有一个中断控制器(GIC),众多的外部设备的中断引脚会连接到中断控制器,由中断控制器负责中断优先级调度,然后发送中断信号给ARM处理。
![](image-1.png)
外设中发生了重要的事情之后,需要通知处理器优先解决,中断发生的时刻与当前正在执行的指令无关,因此中断的发生时间点是异步的。此外，SError（System Error) 也是ARM架构中的一种特殊中断，用于报告系统级别的错误。它主要用于处理与硬件相关的错误，尤其是在虚拟化和安全上下文中。

# 3.2 中止(Abort)
中止主要有指令中止(instruction abort)和数据中止(data abort)两种。中止通常是指**访问内存地址**时发生了错误(如缺页等),处理器内部的MMU捕获这些错请误并且报告给处理器。指令中止是指当处理器尝试执行某条指令时发生了错误, 而数据中止是指使用加载或者存储指令读写外部存储单元时发生了错误。

# 3.3 复位(Reset)
reset是优先级最高的异常,且无法mask屏蔽。复位操作通常用于让CPU复位引脚产生复位信号,让CPU进入复位状态,并重新启动。reset有两种,一种是cold reset, 另外一种是warm reset,它们之间可唯一的不同是是否reset cpu core build-in的debug HW block(调试硬件模块)。

# 3.4 系统调用(System Call)
主要作用：软件主动通过特殊指令请求更高异常等级的程序所提供服务。

1）SVC指令, Supervisor Call 用于EL0（user mode）的软件用来申请EL1（OS service）软件的服务。
2）HVC指令, Hypervisor Call 用于guest OS用来请求hypervisor的服务。
3）SMC指令, Secure monitor Call 用与normal world与secure world间的切换
![](image-2.png)

# 4.异步异常和同步异常
虽然异常各具形态，但是基本可以分成两类，一类是Asynchronous exception，另外一类是Synchronous exception。
# 4.1 异步异常
异步异常是指异常触发的原因与处理器当前正在执行的指令无关的异常。
异步异常特点：
1. 异常和CPU执行的指令无关。
2. 返回地址是硬件保存下来并提供给handler，以便进行异常返回现场的处理。
常见的异步异常包括物理中断和虚拟中断：
* 物理中断分为3种：SError、IRQ、FIQ。
* 虚拟中断分为3种：vSError、vIRQ、vFIQ。

# 4.2 同步异常
同步异常是指处理器执行某条指令而直接导致的异常,往往需要在异常处理函数里处理该异常之后,处理器才能继续执行。
同步异常特点：
1）异常的产生是和cpu core执行的指令或者试图执行执行相关
2）硬件提供给handler的返回地址就是产生异常的那一条指令所在的地址
常见的同步异常：
* 尝试访问一个异常等级不恰当的寄存器。
* 尝试执行关闭或者没有定义(undefined)的指令。
* 使用没有对齐的SP。
* 尝试执行与PC指针没有对齐的指令。
* 软件产生的异常,如执行SVC、HVC或SMC指令。
* 地址翻译或者权限等导致的数据异常。
* 地址翻译或者权限等导致的指令异常。
* 调试导致的异常,如断点异常、观察点异常、软件单步异常等。

# 5. 异常处理流程
# 5.1 异常发生
当发生异常时，处理器会保存PE的当前状态以及异常返回地址，然后进入特定模式来处理异常
CPU会做以下事情：
1. 保存当前处理器状态，把PSTATE(Processor Status Register)的值保存到对应异常等级的SPSR_ELx中(Saved Program Status Register)，每个异常级别都有一个 SPSR，EL0除外，即 SPSR_ELx。发生异常时，使用的 SPSR_ELx 对应于发生异常的异常级别。
2. 把返回地址保存在对应目标异常等级的ELR_ELx中(Exception Link Register for Exception Level), ELR在ARM处理器中用于记录异常发生时的指令地址，确保在处理完异常后可以恢复到正确的执行点。
3. 把PSTATE寄存器里的D、A、I、F标志位都设置为1, 相当于把调试异常、SError、IRQ以及FIQ都关闭。
   + D - Debug exception mask bit
   + A - SError asynchronous exception mask bit, for example, asynchronous external abort
   + I - IRQ asynchronous exception mask bit
   + F - FIQ asynchronous exception mask bit
4. 对同步异常, 异常原因会被写入ESR_ELx(Exception Syndrome Register for Exception Level, 异常综合寄存器)。对于与地址相关的同步异常，例如 MMU 故障，触发异常的虚拟地址被写入故障地址寄存器 FAR_ELx(Fault Address Register)。
5. 切换SP(Stack Pointer)寄存器为目标异常等级的SP_Elx或者SP_EL0寄存器。
6. 从异常发生现场的异常等级切换到对应目标异常等级,然后跳转到异常向量表里。

# 5.2 异常返回
一旦异常处理程序处理完异常，处理程序就会返回到异常发生之前运行的代码,通过执行一条ERET指令即可从异常返回，此指令会完成以下工作：
1. 从ELR_EL x中恢复PC指针。
2. 从SPSR_ELX中恢复PSTATE寄存器的状态。
![](image-3.png)

# 6. 异常向量表
# 6.1 向量表说明
发生异常时，处理器必须执行与该异常对应的处理程序代码。内存中存储处理程序的位置称为异常向量。在ARM架构中，异常向量存储在一个表中，称为异常向量表。
每个异常级别都有自己的向量表，也就是说，EL3、EL2 和 EL1 各有一个向量表。该表包含要执行的指令，而不是一组地址。这些通常是将内核引导至完整异常处理程序的分支指令
关于向量表的跳转与以下几个条目相关：
* 基地址由 VBAR_ELn：指定异常向量表的基地址
* 异常类型（SError、FIQ、IRQ 或同步）
* 同一个异常级别发生异常，要使用的堆栈指针（SP0 或 SPn）
* 较低的异常级别发生异常的，则为下一个较低级别（AArch64 或 AArch32）的执行状态

![](image-4.png)

跳转举例：
例1:
处理器在内核态(EL1异常等级)中触发了IRQ, 并且系统通过配置SPSEL(Stack Pointer Select Registe)寄存器来使用SP_ELx寄存器作为栈指针,处理器会跳转到"VBAR_EL1+0x280"地址处的异常向量中。
如果系统通过配置SPSEL寄存器来使用SP_EL0寄存器作为栈指针,那么处理器会跳转到"VBAR_EL1+0x80"地址处的异常向量中(EL0非特权模式，不处理异常)。

例2:
处理器在用户态(EL0)执行时触发了IRQ,假设用户态的执行状态为AArch64并且该异常会陷入EL1中,那么处理器会跳转到"VBAR_EL1+0x480"地址处的异常向量中。
假设用户态的执行状态为AArch32并且该异常会陷入EL1中,那么处理器会跳转到"VBAR EL1+0x680"地址处的异常向量中。

## 6.2 linux 内核异常向量表代码（v5.19.17）
```C
/*
 * Exception vectors.
 */
	.pushsection ".entry.text", "ax"

	.align	11
SYM_CODE_START(vectors)
	kernel_ventry	1, t, 64, sync		// Synchronous EL1t
	kernel_ventry	1, t, 64, irq		// IRQ EL1t
	kernel_ventry	1, t, 64, fiq		// FIQ EL1h
	kernel_ventry	1, t, 64, error		// Error EL1t

	kernel_ventry	1, h, 64, sync		// Synchronous EL1h
	kernel_ventry	1, h, 64, irq		// IRQ EL1h
	kernel_ventry	1, h, 64, fiq		// FIQ EL1h
	kernel_ventry	1, h, 64, error		// Error EL1h

	kernel_ventry	0, t, 64, sync		// Synchronous 64-bit EL0
	kernel_ventry	0, t, 64, irq		// IRQ 64-bit EL0
	kernel_ventry	0, t, 64, fiq		// FIQ 64-bit EL0
	kernel_ventry	0, t, 64, error		// Error 64-bit EL0

	kernel_ventry	0, t, 32, sync		// Synchronous 32-bit EL0
	kernel_ventry	0, t, 32, irq		// IRQ 32-bit EL0
	kernel_ventry	0, t, 32, fiq		// FIQ 32-bit EL0
	kernel_ventry	0, t, 32, error		// Error 32-bit EL0
SYM_CODE_END(vectors)
```
# 7. 总结
Linux中的异常处理机制是操作系统稳定性、安全性和性能的关键组成部分。
通过有效地管理和处理各种异常情况，Linux能够提供健壮的运行环境，支持高效的多任务处理和进程间通信。
异常处理不仅保障了系统的正常运行，也为开发人员提供了必要的工具来维护和优化系统。