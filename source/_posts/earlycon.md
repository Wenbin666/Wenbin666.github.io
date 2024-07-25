---
title: linux kernel earlycon 梳理
date: 2023-04-18 20:50:44
tags: earlycon
categories:  
  - kernel
---

# linux kernel earlycon 梳理

# 1. earlycon 简介
**背景**：在kernel启动早期，还没有为串口等设备等注册console，此时无法通过正常的console来输出log。
linux提供了early console（earlycon）机制，用于实现早期log的输出，对应console也称为boot console，目前存在早期的early_printk和后期的earlycon两种方式：
* early_printk：平台主要实现early_console->write的adduart函数，各个平台通过函数定义的方式来进行实现，兼容性较差，并且uart寄存器地址都是由自己定义，和正常console的定义完全独立开来，可移植性较差。
* earlycon：通过__earlycon_table维护所有的earlycon_id，通过dts中的正常console的compatible获取到所需要使用的earlycon_id，兼容性较好。而且earlycon可以通过后续真正使用的设备树dts获取正常console使用的uart寄存器地址，来作为earlycon write实现中的uart寄存器基址，可移植性较好。
本文以兼容性较好的earlycon进行分析：

# 2. earylycon 实现关键流程
* kernel中打开对应的宏： CONFIG_SERIAL_EARLYCON
* dts添加相应节点：earlycon= xxx
* 实现一个early console的setup接口，并调用serial_core.h中提供的注册接口将其注册到kernel中
* 实现具体的wrte函数，用于log输出

## 2.1 earlycon注册方式
```C
#define EARLYCON_DECLARE(_name, fn) OF_EARLYCON_DECLARE(_name, "", fn)
#define OF_EARLYCON_DECLARE(_name, compat, fn)              \
      _OF_EARLYCON_DECLARE(_name, compat, fn,             \
                   __UNIQUE_ID(__earlycon_##_name))

#define _OF_EARLYCON_DECLARE(_name, compat, fn, unique_id)      \
      static const struct earlycon_id unique_id           \
           EARLYCON_USED_OR_UNUSED __initconst            \
          = { .name = __stringify(_name),             \
              .compatible = compat,               \
              .setup = fn  };                 \
      static const struct earlycon_id EARLYCON_USED_OR_UNUSED     \
          __section("__earlycon_table")               \
          * const __PASTE(__p, unique_id) = &unique_id
```
通过EARLYCON_DECLARE 或 OF_EARLYCON_DECLARE进行earlycon注册，参数主要包含early console的名称、开始使用之前的初始化setup接口和设备树compatible参数。

相关注册示例：
```C
amba-pl011.c:2637:OF_EARLYCON_DECLARE(pl011, "arm,pl011", pl011_early_console_setup);
altera_uart.c:527:OF_EARLYCON_DECLARE(uart, "altr,uart-1.0", altera_uart_earlycon_setup);
altera_jtaguart.c:402:OF_EARLYCON_DECLARE(juart, "altr,juart-1.0", altera_jtaguart_earlycon_setup);
rda-uart.c:695:OF_EARLYCON_DECLARE(rda, "rda,8810pl-uart",
```
# 3. 关键数据结构和函数
## 3.1 earlycon_device
```C
struct earlycon_device {
    struct console *con;
    struct uart_port port;
    char options[16];       /* 如115200n8 */
    unsigned int baud;
};
```
struct console 控制台结构体，在[consle文章](https://wenbin666.github.io/2024/07/15/console/)已经说明
struct uart_port 是抽象虚拟的串口设备结构体

## 3.2 earlycon_id
```C
struct earlycon_id {
    char    name[15];
    char    name_term;  /* In case compiler didn't '\0' term name */
    char    compatible[128];
    int (*setup)(struct earlycon_device *, const char *options);
};
```
• name :earlyconle的名字，最终会作为相应的console名称
• compatible : 用于匹配uart对应的dts node
• setup : 用来为earlycon设置write函数

## 3.3 函数分析
以pl011驱动代码进行分析：
### 3.3.1 基本寄存器读写实现
读寄存器：
```C
static unsigned int pl011_read(const struct uart_amba_port *uap,
			       unsigned int reg)
{
	void __iomem *addr = uap->port.membase + pl011_reg_to_offset(uap, reg);

	return (uap->port.iotype == UPIO_MEM32) ?
		readl_relaxed(addr) : readw_relaxed(addr);
}
```
写寄存器：
```C
static void pl011_write(unsigned int val, const struct uart_amba_port *uap,
			unsigned int reg)
{
	void __iomem *addr = uap->port.membase + pl011_reg_to_offset(uap, reg);

	if (uap->port.iotype == UPIO_MEM32)
		writel_relaxed(val, addr);
	else
		writew_relaxed(val, addr);
}
```
### 3.3.2 setup函数
```C
static int pl011_console_setup(struct console *co, char *options)
{
	struct uart_amba_port *uap;
	int baud = 38400;
	int bits = 8;
	int parity = 'n';
	int flow = 'n';
	int ret;

	// 入参检查
	if (co->index >= UART_NR)
		co->index = 0;
	uap = amba_ports[co->index];
	if (!uap)
		return -ENODEV;

	// pinctrl和电源相关使能
	pinctrl_pm_select_default_state(uap->port.dev);
	// clk相关
	ret = clk_prepare(uap->clk);
	if (ret)
		return ret;

	if (dev_get_platdata(uap->port.dev)) {
		struct amba_pl011_data *plat;

		plat = dev_get_platdata(uap->port.dev);
		if (plat->init)
			plat->init();
	}

	uap->port.uartclk = clk_get_rate(uap->clk);

	if (uap->vendor->fixed_options) {
		baud = uap->fixed_baud;
	} else {
		if (options)
			uart_parse_options(options,
					   &baud, &parity, &bits, &flow);
		else
			pl011_console_get_options(uap, &baud, &parity, &bits);
	}
    //根据实际的参数去配置底层寄存器，完全uart初始化
	return uart_set_options(&uap->port, co, baud, parity, bits, flow);
}
static int __init pl011_early_console_setup(struct earlycon_device *device,
					    const char *opt)
{
	if (!device->port.membase)
		return -ENODEV;

	device->con->write = pl011_early_write;
	device->con->read = pl011_early_read;

	return 0;
}
```

### 3.3.3 write函数实现
```C
static void pl011_early_write(struct console *con, const char *s, unsigned int n)
{
	struct earlycon_device *dev = con->data;

	uart_console_write(&dev->port, s, n, pl011_putc);
}
```

# 4.earlycon 源码调用关系
```C
start_kernel()
 -> parse_early_param()
 (early_param("earlycon", param_setup_earlycon)通过注册接口过渡)
  -> param_setup_earlycon()  //tty/serial/earlycon.c
   -> early_init_dt_scan_chosen_stdout()  // drivers/of/fdt.c
   -> of_setup_earlycon(match, offset, options) //具体的setup函数
```
# 5. 类比 uboot debug uart
Uart debug uart 其实和earlycon是一样的东西，不得不说uboot可以算是一个小kernel；
官方说明文档DOC:
```C
/*
 * The debug UART is intended for use very early in U-Boot to debug problems
 * when an ICE or other debug mechanism is not available.
 *
 * To use it you should:
 * - Make sure your UART supports this interface
 * - Enable CONFIG_DEBUG_UART
 * - Enable the CONFIG for your UART to tell it to provide this interface
 *       (e.g. CONFIG_DEBUG_UART_NS16550)
 * - Define the required settings as needed (see below)
 * - Call debug_uart_init() before use
 * - Call printch() to output a character
 *
```
根据说明可以知道：debug uart是在uboot早期用于调试，只需要进行简单的初始化和定义一个输出接口就可以使用，也给出了驱动创建自己debug uart的方法。
```C
_debug_uart_init(void)
_debug_uart_putc(int ch)
```
