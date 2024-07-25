---
title: uboot 引导kernel过程梳理
date: 2024-07-17 20:57:48
tags: uboot
categories:
 - uboot
---
# uboot 引导kernel过程梳理
# 1. 关键流程
## 1.1 功能层面说明
uboot的主要功能就是引导Linux内核启动，从功能性角度说明，流程如下：
1. 初始化硬件设备，包括串口、网卡、Flash等
2. 加载内核镜像文件，并将其解压到内存中
3. 对内核进行校验，确保其完整性
4. 根据引导参数设置内核的配置参数
5. 跳转到内核的入口点执行
## 1.2 函数层面说明
从函数调用的角度进行分析，关键函数主要包括以下几个：
```bash
_start (arch/arm/cpu/armv8/start.S)
 -> _main (arch/arm/lib/ctr0_64.S)
  -> board_init_f (common/board_f.c)
   -> relocate_code (arch/arm/lib/relocate_64.S)
    -> c_runtime_cpu_setup (arch/arm/cpu/armv8/start.S)
     -> board_init_r (common/board_r.c)
```
## 1.3 函数层面分析
### 1.3.1 _start(start.S)
任何程序的入口函数都是在链接时决定的:
Uboot下可以通过在config文件中配置CONFIG_SYS_LDSCRIPT来指定链接脚本,通过CONFIG_SYS_TEXTBASE指定入口地址
链接器在链接时会查找目标文件中的_start符号代表的地址，把它设置为整个程序的入口地址。
start.S的代码段位于uboot代码段最开始的位置，而_start符号对于ArmV8架构来说则位于arch/arm/cpu/armv8/start.S文件内。
start.S的主要功能入下：
```bash
.globl	_start
_start:
#if defined(CONFIG_LINUX_KERNEL_IMAGE_HEADER)
#include <asm/boot0-linux-kernel-header.h>
#elif defined(CONFIG_ENABLE_ARM_SOC_BOOT0_HOOK)
/*
 * Various SoCs need something special and SoC-specific up front in
 * order to boot, allow them to set that in their boot0.h file and then
 * use it here.
 */
#include <asm/arch/boot0.h>
#else
	b	reset
#endif
```
**CONFIG_LINUX_KERNEL_IMAGE_HEADER** 宏用于配置 Linux 内核镜像的头部信息。
该宏定义了在 Linux 内核编译过程中，是否包含额外的头部信息，例如 ELF 或者其他格式的头部信息。根据宏的设置，内核编译过程中会根据需要添加或移除相应的头部信息。
**CONFIG_ENABLE_ARM_SOC_BOOT0_HOOK** 宏用于启用或禁用 ARM 架构的 SOC芯片的引导钩子（boot0 hook）。
引导钩子是用于在 SOC 芯片引导过程中执行特定代码的一种机制，可用于执行一些预处理或初始化操作。通过配置该宏，可以控制是否启用 SOC 芯片的引导钩子功能。
start 符号后紧跟的第一条指令, 指令为跳转指令，跳转到reset标号处.

```bash
    .align 3

.globl	_TEXT_BASE
_TEXT_BASE:
	.quad	CONFIG_TEXT_BASE

/*
 * These are defined in the linker script.
 */
.globl	_end_ofs
_end_ofs:
	.quad	_end - _start

.globl	_bss_start_ofs
_bss_start_ofs:
	.quad	__bss_start - _start

.globl	_bss_end_ofs
_bss_end_ofs:
	.quad	__bss_end - _start
```
随后，align 3 表示后续符号或数据按照 2 的 3 次方（即 8 字节）对齐，用于高效地访问内存。
.globl _TEXT_BASE 用于将符号 _TEXT_BASE 声明为全局符号，使得该符号能够在整个程序的各个模块中可见和访问。
.quad CONFIG_TEXT_BASE 定义一个双字（64 位，8 字节）大小的符号，其值为 CONFIG_TEXT_BASE 所代表的地址，也就是通过_TEXT_BASE 符号来引用 CONFIG_TEXT_BASE 的地址，方便了在代码中使用该地址。
紧接着定义了几个符号_end_ofs、_bss_start_ofs和_bss_end_otfs
```bash
reset:
	/* Allow the board to save important registers */
	b	save_boot_params
.globl	save_boot_params_ret
save_boot_params_ret:
```
看注释这部分内容是为了储存一些关键的寄存器参数。

```bash
#if CONFIG_POSITION_INDEPENDENT && !defined(CONFIG_SPL_BUILD)
	/* Verify that we're 4K aligned.  */
	adr	x0, _start
	ands	x0, x0, #0xfff
	b.eq	1f
0:
	/*
	 * FATAL, can't continue.
	 * U-Boot needs to be loaded at a 4K aligned address.
	 *
	 * We use ADRP and ADD to load some symbol addresses during startup.
	 * The ADD uses an absolute (non pc-relative) lo12 relocation
	 * thus requiring 4K alignment.
	 */
	wfi
	b	0b
1:

	/*
	 * Fix .rela.dyn relocations. This allows U-Boot to be loaded to and
	 * executed at a different address than it was linked at.
	 */
pie_fixup:
	adr	x0, _start		/* x0 <- Runtime value of _start */
	ldr	x1, _TEXT_BASE		/* x1 <- Linked value of _start */
	subs	x9, x0, x1		/* x9 <- Run-vs-link offset */
	beq	pie_fixup_done
	adrp    x2, __rel_dyn_start     /* x2 <- Runtime &__rel_dyn_start */
	add     x2, x2, #:lo12:__rel_dyn_start
	adrp    x3, __rel_dyn_end       /* x3 <- Runtime &__rel_dyn_end */
	add     x3, x3, #:lo12:__rel_dyn_end
pie_fix_loop:
	ldp	x0, x1, [x2], #16	/* (x0, x1) <- (Link location, fixup) */
	ldr	x4, [x2], #8		/* x4 <- addend */
	cmp	w1, #1027		/* relative fixup? */
	bne	pie_skip_reloc
	/* relative fix: store addend plus offset at dest location */
	add	x0, x0, x9
	add	x4, x4, x9
	str	x4, [x0]
pie_skip_reloc:
	cmp	x2, x3
	b.lo	pie_fix_loop
pie_fixup_done:
#endif
```
如果Uboot配置了支持位置独立可执行（Position Independent Executable，PIE）的情况下，且没有构建SPL，会执行以上代码。
当开启 CONFIG_POSITION_INDEPENDENT 选项时，Linux 内核将会启用 PIE 模式，在编译内核时，会生成支持位置独立可执行格式的二进制文件。
这可以提高系统的安全性，减少受到针对特定内存地址的攻击的风险。然而，启用这个选项可能会稍微增加程序的运行时开销和内存占用，因为需要进行额外的重定位操作。

```bash
#if defined(CONFIG_ARMV8_SPL_EXCEPTION_VECTORS) || !defined(CONFIG_SPL_BUILD)
.macro	set_vbar, regname, reg
	msr	\regname, \reg
.endm
	adr	x0, vectors
#else
.macro	set_vbar, regname, reg
.endm
#endif
```
这段代码，根据不同的配置选项，动态定义了一个设置异常向量表地址寄存器的宏。
在 ARMv8 架构下或者非 SPL 构建时，使用宏 set_vbar 来设置异常向量表地址；在其他情况下，set_vbar 宏为空，不执行任何操作。

```bash
	/*
	 * Could be EL3/EL2/EL1, Initial State:
	 * Little Endian, MMU Disabled, i/dCache Disabled
	 */
	switch_el x1, 3f, 2f, 1f
3:	set_vbar vbar_el3, x0
	mrs	x0, scr_el3
	orr	x0, x0, #0xf			/* SCR_EL3.NS|IRQ|FIQ|EA */
	msr	scr_el3, x0
	msr	cptr_el3, xzr			/* Enable FP/SIMD */
	b	0f
2:	mrs	x1, hcr_el2
	tbnz	x1, #HCR_EL2_E2H_BIT, 1f	/* HCR_EL2.E2H */
	orr	x1, x1, #HCR_EL2_AMO_EL2	/* Route SErrors to EL2 */
	msr	hcr_el2, x1
	set_vbar vbar_el2, x0
	mov	x0, #0x33ff
	msr	cptr_el2, x0			/* Enable FP/SIMD */
	b	0f
1:	set_vbar vbar_el1, x0
	mov	x0, #3 << 20
	msr	cpacr_el1, x0			/* Enable FP/SIMD */
0:
	msr	daifclr, #0x4			/* Unmask SError interrupts */
```
首先是判断异常等级，是EL3、EL2还是EL1,其中switch_el的宏定义如下：
```bash
/*
 * Branch according to exception level
 */
.macro  switch_el, xreg, el3_label, el2_label, el1_label
    mrs \xreg, CurrentEL
    cmp \xreg, 0xc
    b.eq    \el3_label
    cmp \xreg, 0x8
    b.eq    \el2_label
    cmp \xreg, 0x4
    b.eq    \el1_label
.endm
```
其中.macro和endm是成对出现，用于定义宏。
代码首先将CurrentEL寄存器的值读取到通用寄存器中,随后分别和0xc、0x8、以及0x4比较,若等于0xc则跳转到EL3,若等于0x8则跳到EL2处执行,若等于0x4则跳转到EL1处执行。
查看看Armv8寄存器手册,CurrentEL具体定义如下：
![](image.png)
可以看到,Bit3:2定义了当前异常等级,故0xc对应EL3,0x8对应EL2,0x4对应EL1，如果是热启动，默认的异常等级为EL3，则执行以下代码：
```bash
3:	set_vbar vbar_el3, x0
	mrs	x0, scr_el3
	orr	x0, x0, #0xf			/* SCR_EL3.NS|IRQ|FIQ|EA */
	msr	scr_el3, x0
	msr	cptr_el3, xzr			/* Enable FP/SIMD */
	b	0f
```
![](image-1.png)
SCR_EL3为安全配置寄存器，主要用于定义安全状态和IRQ、FIQ、SError等。
Bit[0]:No-Secure bit. 与SCR_EL3.NSE 共同决定EL2的安全状态和更低的异常等级。
![alt text](image-2.png)
Bit[1]：IRQ bit，Physical IRQ Routing. 当配置为1时，路由中断到EL3.
Bib[2]: FIQ bit, Physical FIQ Routing. 当配置为1时，FIQ中断会被路由到EL3.
Bit[3]: External Abort and SError exception routing. 外部中止和SError(系统错误)异常路由。
CPTR_EL3, Architectural Feature Trap Register(架构特性陷阱寄存器)
bit[10]: 捕获访问高级SIMD和浮点功能的指令的执行，从所有异常级别、任何安全状态和两个执行状态一直到EL3。
详细细节分析：TBD
最终：跳转到_main
```bash
bl main
```

### 1.3.2 _main(crt0_64.S)
主要功能：负责U-Boot启动过程中与目标target无关的阶段
```bash
/*
 * This file handles the target-independent stages of the U-Boot
 * start-up where a C runtime environment is needed. Its entry point
 * is _main and is branched into from the target's start.S file.
 *
 * _main execution sequence is:
 *
 * 1. Set up initial environment for calling board_init_f().
 *    This environment only provides a stack and a place to store
 *    the GD ('global data') structure, both located in some readily
 *    available RAM (SRAM, locked cache...). In this context, VARIABLE
 *    global data, initialized or not (BSS), are UNAVAILABLE; only
 *    CONSTANT initialized data are available. GD should be zeroed
 *    before board_init_f() is called.
 *
 * 2. Call board_init_f(). This function prepares the hardware for
 *    execution from system RAM (DRAM, DDR...) As system RAM may not
 *    be available yet, , board_init_f() must use the current GD to
 *    store any data which must be passed on to later stages. These
 *    data include the relocation destination, the future stack, and
 *    the future GD location.
 *
 * 3. Set up intermediate environment where the stack and GD are the
 *    ones allocated by board_init_f() in system RAM, but BSS and
 *    initialized non-const data are still not available.
 *
 * 4a.For U-Boot proper (not SPL), call relocate_code(). This function
 *    relocates U-Boot from its current location into the relocation
 *    destination computed by board_init_f().
 *
 * 4b.For SPL, board_init_f() just returns (to crt0). There is no
 *    code relocation in SPL.
 *
 * 5. Set up final environment for calling board_init_r(). This
 *    environment has BSS (initialized to 0), initialized non-const
 *    data (initialized to their intended value), and stack in system
 *    RAM (for SPL moving the stack and GD into RAM is optional - see
 *    CONFIG_SPL_STACK_R). GD has retained values set by board_init_f().
 *
 * TODO: For SPL, implement stack relocation on AArch64.
 *
 * 6. For U-Boot proper (not SPL), some CPUs have some work left to do
 *    at this point regarding memory, so call c_runtime_cpu_setup.
 *
 * 7. Branch to board_init_r().
 *
 * For more information see 'Board Initialisation Flow in README.
 */
```
### 1.3.3 board_init_f(uboot/common/board_f.c)
board_init_f 函数执行一些早期的初始化，其依次执行数组init_sequence_f中的每个函数
**数组 init_sequence_f**
```bash
static const init_fnc_t init_sequence_f[] = {
	setup_mon_len,
	initf_malloc,
	log_init,
	initf_bootstage,	/* uses its own timer, so does not need DM */
	event_init,
	bloblist_maybe_init,
	setup_spl_handoff,
	INITCALL_EVENT(EVT_FSP_INIT_F),
	arch_cpu_init,		/* basic arch cpu dependent setup */
	mach_cpu_init,		/* SoC/machine dependent CPU setup */
	initf_dm,
	env_init,		/* initialize environment */
	init_baud_rate,		/* initialze baudrate settings */
	serial_init,		/* serial communications setup */
	console_init_f,		/* stage 1 init of console */
	display_options,	/* say that we are here */
	display_text_info,	/* show debugging info if required */
	checkcpu,
	INIT_FUNC_WATCHDOG_INIT
	INITCALL_EVENT(EVT_MISC_INIT_F),
	INIT_FUNC_WATCHDOG_RESET
	announce_dram_init,
	dram_init,		/* configure available RAM banks */
	INIT_FUNC_WATCHDOG_RESET
	reserve_round_4k,
	setup_relocaddr_from_bloblist,
	arch_reserve_mmu,
	reserve_video,
	reserve_trace,
	reserve_uboot,
	reserve_malloc,
	reserve_board,
	reserve_global_data,
	reserve_fdt,
	reserve_bootstage,
	reserve_bloblist,
	reserve_arch,
	reserve_stacks,
	dram_init_banksize,
	show_dram_config,
	INIT_FUNC_WATCHDOG_RESET
	setup_bdinfo,
	display_new_sp,
	INIT_FUNC_WATCHDOG_RESET
	reloc_bootstage,
	reloc_bloblist,
	setup_reloc,
	clear_bss,
	NULL,
	...
};
```
### 1.3.4 relocate_code
主要功能：将U-Boot code从其当前位置重新定位到由board_init_f()计算的位置
U-Boot的relocation（重定位）是指将程序中的符号地址从编译时的地址转换为运行时的地址。
在U-Boot启动过程中，链接器会将所有的目标文件合并成一个可执行文件，并生成符号表。
符号表是一个关键的数据结构，它记录了程序中所有的符号（变量、函数等）的名称和地址。在程序执行时，这些符号需要被正确解析和跳转，以实现程序的正确执行。

### 1.3.5 c_runtime_cpu_setup
作用：对于U-Boot的正常工作，一些CPU在内存方面还有一些工作(重定位中断向量表) 要做，所以调用c_runtime_cpu_setup。
CPU运行时环境是指在程序运行期间，与CPU相关的一些配置和管理操作。

### 1.3.6 board_init_r(uboot/common/board_r.c)
board_init_r也有一个函数数组init_sequence_r，依次执行init_sequence_r中的函数
**init_sequence_r**
```bash
static init_fnc_t init_sequence_r[] = {
	initr_trace,
	initr_reloc,
	event_init,
	/* TODO: could x86/PPC have this also perhaps? */
	initr_reloc_global_data,
	initr_barrier,
	initr_malloc,
	log_init,
	initr_bootstage,	/* Needs malloc() but has its own timer */
	/*
	 * TODO: printing of the clock inforamtion of the board is now
	 * implemented as part of bdinfo command. Currently only support for
	 * davinci SOC's is added. Remove this check once all the board
	 * implement this.
	 */
	initr_dm_devices,
	stdio_init_tables,
	serial_initialize,
	initr_announce,
	dm_announce,
	INIT_FUNC_WATCHDOG_RESET
	arch_initr_trap,
	INIT_FUNC_WATCHDOG_RESET
	INIT_FUNC_WATCHDOG_RESET
	stdio_add_devices,
	jumptable_init,
	INIT_FUNC_WATCHDOG_RESET
	INITCALL_EVENT(EVT_LAST_STAGE_INIT),
	run_main_loop,
	...
};
```
### 1.3.7 run_main_loop
```bash
run_main_loop
 -> cli_init
 -> run_preboot_environment_command
    -> run_command_list
 -> bootdelay_process
 -> cli_loop
```
### 1.3.8 总结
![](image-3.png)

### 1.3.9 Uboot autoboot功能
u-boot autoboot功能简介（uboot/doc/README.autoboot）
具体配置：
1）可以配置是否在autoboot之前等待一段时间以便让用户输入从而进入命令行：通过CONFIG_BOOTDELAY配置
2）通过CONFIG_BOOTCOMMAND配置boot kernel所使用的命令
3）通过CONFIG_BOOTARGS配置命令行参数

### 1.3.10 booti/bootm/bootz
功能：均为U-Boot中用于引导内核启动的命令，但它们在使用和功能上有所区别。
bootm(boot multi)，启动多个内核。它从内存中的某个地址加载内核，并在启动后将控制权交给内核。bootm命令会检查内核镜像的有效性，如果内核镜像是压缩的，它会自动解压缩。
booti(boot image)，用于启动ARM架构的ARM64内核。与bootm类似，booti也会检查内核镜像的有效性，并自动解压缩压缩的内核镜像。
bootz(boot zImage), 用于启动zImage内核。与bootm和booti不同的是，bootz命令不进行任何解压缩操作，而是直接从内存中的某个地址加载内核镜像。因此，bootz需要内核镜像已经被解压缩并放置在正确的内存地址中。
代码层面分析：
![](image-4.png)
总结：最终都跳转到do_bootm_states函数

# 2. do_bootm_states细节
## 2.1 整体梳理
这这部分内容总体上实现了一个类似状态机的机制，将整个过程分成很多个阶段，每个阶段可以称为subcommand.
u-boot/include/image.h
```bash
#define BOOTM_STATE_START	0x00000001
#define BOOTM_STATE_FINDOS	0x00000002
#define BOOTM_STATE_FINDOTHER	0x00000004
#define BOOTM_STATE_LOADOS	0x00000008
#define BOOTM_STATE_RAMDISK	0x00000010
#define BOOTM_STATE_FDT		0x00000020
#define BOOTM_STATE_OS_CMDLINE	0x00000040
#define BOOTM_STATE_OS_BD_T	0x00000080
#define BOOTM_STATE_OS_PREP	0x00000100
#define BOOTM_STATE_OS_FAKE_GO	0x00000200	/* 'Almost' run the OS */
#define BOOTM_STATE_OS_GO	0x00000400
#define BOOTM_STATE_PRE_LOAD	0x00000800
#define BOOTM_STATE_MEASURE	0x00001000
```
总体函数调用关系梳理：
```C
do_bootm_states
-> bootm_start             //该函数用于清空images结构体,初始化引导加载程序的一些变量
-> bootm_findos            //搜索存储介质（如闪存、硬盘）中的内核镜像，并获取其地址和大小信息
-> bootm_find_other        //查找其他可引导的操作系统内核镜像
-> bootm_load_os           //将内核镜像从存储介质中复制到内存中，以便后续的启动操作
-> bootm_os_get_boot_func  //根据系统类型返回相应的启动函数，用于执行操作系统的启动
-> boot_fn                 //根据传入的不同参数执行相应的操作，例如设置跳转地址、调用系统启动函数等
-> boot_selected_os        //调用相应的系统启动函数并传递必要的参数，以启动选定的操作系统
 -> boot_os                //启动特定的操作系统
  -> do_bootm_linux        //执行Linux内核的启动过程
   -> boot_prep_linux      //处理环境变量bootargs，将传递给Linux内核的参数保存到环境变量中
   -> boot_jump_linux      //跳转到内核的启动点,将控制权传递给内核，从而开始操作系统的启动
```
关键流程提炼：
![](image-5.png)
## 2.2 主要函数梳理
### 2.2.1 bootm_start
作用：
1.	初始化全局变量images
2.	初始化lmb(logic memory block，逻辑内存块，在引导初期，泛用内核内存分配器还没有开始工作时对内存区域进行管理的方法之一)。
3.	bootstage_mark_name标志当前状态为bootm_start

### 2.2.2 bootm_findos
作用：
1.	获取到kernel被放到了哪个位置、获取内核镜像header、起始地址和长度
2.	根据image的hdr来做crc，获取一些基本的os信息到images结构体中

### 2.2.3 bootm_findother
作用：
1.	尝试加载可用的ramdisk
2.	flattened device tree（获取设备树地址）
3.	specifically marked "loadable" images (loadables are FIT only)

### 2.2.4 bootm_load_os
bootm_load_os 又可以细分为两个部分：boot_prep_linux 和 boot_jump_linux

#### 2.2.4.1 boot_prep_linux
boot_prep_linux 函数的主要功能是为 Linux 内核的启动过程准备必要的环境和参数，并确保内核能够从正确的状态和配置启动.
这些准备工作有助于保证 Linux 内核的正常启动和运行。

#### 2.2.4.2 boot_jump_linux
调用kernel_entry，参数1为0，参数2为machid，参数3为bi_boot_params。

至此uboot的使命已经完成，后续引导起来的kernel将继续执行！
