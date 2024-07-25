---
title: linux kernel console 建立过程
date: 2024-07-15 20:39:09
tags: console
categories:  
  - kernel
---
# linux kernel console 建立过程
以kernel v5.19.17版本进行分析

# 1. console_init
```C
/* include/asm-generic/vmlinux.lds.h */
#define CON_INITCALL							\
		__con_initcall_start = .;				\
		KEEP(*(.con_initcall.init))				\
		__con_initcall_end = .;
```
在链接文件分配SECTIONS
```C
/* include/linux/init.h */
#define console_initcall(fn)	___define_initcall(fn, con, .con_initcall)
```
通过 console_initcall（fn)定义的fn函数都会被放到.con_initcal.init段里面。
这个段的开始是 con_initcall_start, 结束是 __con_initcall_end。
console_init函数在start_kernel中被调用到

```C
/* kernel/init/main.c */
start_kernel
 -> console_init
```
console_init在printk.c中被定义

```C
void __init console_init(void)
{
	int ret;
	initcall_t call;
	initcall_entry_t *ce;

	/* Setup the default TTY line discipline. */
	n_tty_init(); //初始化tty层

	/*
	 * set up the console device so that later boot sequences can
	 * inform about problems etc..
	 */
	ce = __con_initcall_start;
	trace_initcall_level("console");
	while (ce < __con_initcall_end) {
		call = initcall_from_entry(ce);
		trace_initcall_start(call);
		ret = call();
		trace_initcall_finish(call, ret);
		ce++;
	}
}
```
具体的设备可以通过console_initcall(xxx_console_init)接口来初始化使用的console

# 2. register_console
console结构体如下(既包含变量也包含相应的函数)：
```C
struct console {
	char	name[16];
	void	(*write)(struct console *, const char *, unsigned);
	int	(*read)(struct console *, char *, unsigned);
	struct tty_driver *(*device)(struct console *, int *);  //对应的tty驱动
	void	(*unblank)(void); //取消控制台设备的屏幕休眠
	int	(*setup)(struct console *, char *);
	int	(*exit)(struct console *);
	int	(*match)(struct console *, char *name, int idx, char *options);
	short	flags;
	short	index;
	int	cflag;
	uint	ispeed;
	uint	ospeed;
	u64	seq;
	unsigned long dropped;  //丢失的数据
	void	*data;
	struct	 console *next; // 指向下一个console设备，console被串在一条链上
};
```
具体的register_console函数也在printk.c中
```C
void register_console(struct console *newcon)
{
	struct console *con;
	bool bootcon_enabled = false;
	bool realcon_enabled = false;
	int err;

	for_each_console(con) {
		if (WARN(con == newcon, "console '%s%d' already registered\n",
					 con->name, con->index))
			return;
	} // 如果已经注册过给出提示，并返回

	for_each_console(con) {
		if (con->flags & CON_BOOT)
			bootcon_enabled = true; /* 如果标志位为引导控制台 */
		else
			realcon_enabled = true;
	}

	/* Do not register boot consoles when there already is a real one. */
	if (newcon->flags & CON_BOOT && realcon_enabled) {
		pr_info("Too late to register bootconsole %s%d\n",
			newcon->name, newcon->index);
		return;
	}
    /* 之前已经有了引导console， 现在又配置为引导console，其实是不需要的 ，推测boot console 只有一个*/
	/*
	 * See if we want to enable this console driver by default.
	 *
	 * Nope when a console is preferred by the command line, device
	 * tree, or SPCR.
	 * 第一个使用tty绑定（驱动程序）的真正控制台成为真正的console
	 * The first real console with tty binding (driver) wins. More
	 * consoles might get enabled before the right one is found.
	 *
	 * Note that a console with tty binding will have CON_CONSDEV
	 * flag set and will be first in the list.
	 */
	if (preferred_console < 0) {
		if (!console_drivers || !console_drivers->device ||
		    console_drivers->flags & CON_BOOT) {
			try_enable_default_console(newcon);
		}
	}

	/* See if this console matches one we selected on the command line */
	err = try_enable_preferred_console(newcon, true);

	/* If not, try to match against the platform default(s) */
	if (err == -ENOENT)
		err = try_enable_preferred_console(newcon, false);

	/* printk() messages are not printed to the Braille console. */
	if (err || newcon->flags & CON_BRL)
		return;

	/*
	 * If we have a bootconsole, and are switching to a real console,
	 * don't print everything out again, since when the boot console, and
	 * the real console are the same physical device, it's annoying to
	 * see the beginning boot messages twice
	 */
	if (bootcon_enabled &&
	    ((newcon->flags & (CON_CONSDEV | CON_BOOT)) == CON_CONSDEV)) {
		newcon->flags &= ~CON_PRINTBUFFER;
	}

	/*
	 *	Put this console in the list - keep the
	 *	preferred driver at the head of the list.
	 */
	console_lock();
	if ((newcon->flags & CON_CONSDEV) || console_drivers == NULL) {
		newcon->next = console_drivers;
		console_drivers = newcon;
		if (newcon->next)
			newcon->next->flags &= ~CON_CONSDEV;
		/* Ensure this flag is always set for the head of the list */
		newcon->flags |= CON_CONSDEV;
	} else {
		newcon->next = console_drivers->next;
		console_drivers->next = newcon;
	}

	if (newcon->flags & CON_EXTENDED)
		nr_ext_console_drivers++;

	newcon->dropped = 0;
	if (newcon->flags & CON_PRINTBUFFER) {
		/* Get a consistent copy of @syslog_seq. */
		mutex_lock(&syslog_lock);
		newcon->seq = syslog_seq;
		mutex_unlock(&syslog_lock);
	} else {
		/* Begin with next message. */
		newcon->seq = prb_next_seq(prb);
	}
	console_unlock();
	console_sysfs_notify();

	/*
	 * By unregistering the bootconsoles after we enable the real console
	 * we get the "console xxx enabled" message on all the consoles -
	 * boot consoles, real consoles, etc - this is to ensure that end
	 * users know there might be something in the kernel's log buffer that
	 * went to the bootconsole (that they do not see on the real console)
	 */
	con_printk(KERN_INFO, newcon, "enabled\n");
	if (bootcon_enabled &&
	    ((newcon->flags & (CON_CONSDEV | CON_BOOT)) == CON_CONSDEV) &&
	    !keep_bootcon) {
		/* We need to iterate through all boot consoles, to make
		 * sure we print everything out, before we unregister them.
		 */
		for_each_console(con)
			if (con->flags & CON_BOOT)
				unregister_console(con);
	}
}
EXPORT_SYMBOL(register_console);
```
注册console主要是筛选preferred console,放置在全局console drivers链表前面,剩下的console放置链表靠后的位置,并设置相应的flags.

# 3. unregister_console
```C
int unregister_console(struct console *console)
{
	struct console *con;
	int res;

	con_printk(KERN_INFO, console, "disabled\n");

	res = _braille_unregister_console(console);
	if (res < 0)
		return res;
	if (res > 0)
		return 0;

	res = -ENODEV;
	console_lock();
	if (console_drivers == console) {
		console_drivers=console->next;
		res = 0;
	} else {
		for_each_console(con) { //从console的链表中删除
			if (con->next == console) {
				con->next = console->next;
				res = 0;
				break;
			}
		}
	}

	if (res)
		goto out_disable_unlock;

	if (console->flags & CON_EXTENDED)
		nr_ext_console_drivers--;

	/*
	 * If this isn't the last console and it has CON_CONSDEV set, we
	 * need to set it on the next preferred console.
	 */
	if (console_drivers != NULL && console->flags & CON_CONSDEV)
		console_drivers->flags |= CON_CONSDEV;

	console->flags &= ~CON_ENABLED;
	console_unlock();
	console_sysfs_notify();

	if (console->exit)
		res = console->exit(console);

	return res;

out_disable_unlock:
	console->flags &= ~CON_ENABLED;
	console_unlock();

	return res;
}
EXPORT_SYMBOL(unregister_console);
```
早期的earlycon在真正的控制台起来后就是调用这个接口取消注册

# 4. console_setup
```C
start_kernel
-> parse_early_param // 解析早期参数
 -> parse_early_options // 解析早期配置
  -> parse_args
   -> do_early_param
    -> console_setup
```
主要功能是配置console的参数，而这些参数是uboot通过cmdline传递过来的
```C
static int __init console_setup(char *str)
{
	char buf[sizeof(console_cmdline[0].name) + 4]; /* 4 for "ttyS" */
	char *s, *options, *brl_options = NULL;
	int idx;

	/*
	 * console="" or console=null have been suggested as a way to
	 * disable console output. Use ttynull that has been created
	 * for exactly this purpose.
	 */
	if (str[0] == 0 || strcmp(str, "null") == 0) {
		__add_preferred_console("ttynull", 0, NULL, NULL, true);
		return 1;
	}

	if (_braille_console_setup(&str, &brl_options))
		return 1;

	/*
	 * Decode str into name, index, options.
	 */
	if (str[0] >= '0' && str[0] <= '9') {
		strcpy(buf, "ttyS");
		strncpy(buf + 4, str, sizeof(buf) - 5);
	} else {
		strncpy(buf, str, sizeof(buf) - 1);
	}
	buf[sizeof(buf) - 1] = 0;
	options = strchr(str, ',');
	if (options)
		*(options++) = 0;
#ifdef __sparc__
	if (!strcmp(str, "ttya"))
		strcpy(buf, "ttyS0");
	if (!strcmp(str, "ttyb"))
		strcpy(buf, "ttyS1");
#endif
	for (s = buf; *s; s++)
		if (isdigit(*s) || *s == ',')
			break;
	idx = simple_strtoul(s, NULL, 10);
	*s = 0;

	__add_preferred_console(buf, idx, options, brl_options, true);
	return 1;
}
```

# 5. 设备文件与console
从函数层面进行分析
```C
/* kernel/init/main.c */
start_kernel(void)
 -> arch_call_rest_init();
  -> rest_init()
   -> kernel_init()
    -> kernel_init_freeable()
     -> do_basic_setup()
      -> driver_init()
       -> chr_dev_init()
        -> tty_init()
```
tty_init后本质上/dev下面存在tty和console等设备，直到文件系统启动完全，用户可以看到真正的设备。

# 6. tty_io.c下面console_fops
```C
static const struct file_operations console_fops = {
	.llseek		= no_llseek,
	.read_iter	= tty_read,
	.write_iter	= redirected_tty_write,
	.splice_read	= generic_file_splice_read,
	.splice_write	= iter_file_splice_write,
	.poll		= tty_poll,
	.unlocked_ioctl	= tty_ioctl,
	.compat_ioctl	= tty_compat_ioctl,
	.open		= tty_open,
	.release	= tty_release,
	.fasync		= tty_fasync,
};
```
## 6.1 tty_open调用逻辑
```C
tty_open
 -> tty_open_by_driver
  -> tty_lookup_driver
    -> console_device
```
对于一个新实现的输入输出设备，如果想让其作为kernel的printk输出设备，也作为user空间的控制台，需要(基本write，setup的)基础上再实现device方法成员，来返回该设备的tty_driver。

## 6.2 tty_write
```C
tty_write
 -> file_tty_write
  -> do_tty_write
   -> n_tty_write //线路规程
    -> uart_write
     -> port->ops-start_tx //底层驱动实际发送数据
```
console的读写，通过tty_read和tty_write到最后的实际serial设备驱动的读写
## 6.3 tty_read
```C
// 应用层：
tty_read
 -> iterate_tty_read
  -> n_tty_read //线路规程
   -> input_available_p （上层和底层的连接点）
    —> 等待队列开始等待
   // 检查终端设备的可用性及状态，如果没有数据可读，则进行等待
   // 直到满足特定条件，或达到超时时间
   // 根据不同情况返回相应的错误码或继续循环
//底层，从实际硬件的角度进行分析
//中断接收到数据 (console是通过中断触发进行读数据的)
(uart_insert_char ，实际硬件中断有这一层)
 -> tty_insert_flip_char  //放入tty_buffer
  -> tty_flip_buffer_push //数据从tty_buffer推到线路规程
   -> tty_schedule_flip  //并唤醒工作队列
```
注意：没有.read（这是因为通常情况下，终端设备的输入由用户通过键盘输入，而不是由内核直接从终端设备读取。
内核通常通过其他机制来处理终端设备的输入，例如使用中断处理程序或轮询机制来获取用户的输入）

# 7. 总结
## 7.1 参数传递
1.	uboot通过cmdline进行参数传递（console等）
2.	Kernel 通过解析参数获取传递值；
3.	init的时候进行conso_initcall
4.	解析参数的时候调用do_early_param进行console解析，并调用各个console的实际底层console_setup函数
5.	如果没有cmdline，进行默认初始化等配置，如波特率9600

## 7.2 prinkt输出
*	printk会将输出内容添加到一个kernel缓冲区中，叫log_buf
*	log_bu的大小由menuconfig配置
*	printk内容会一直存在log_buf中，若log_buf满了之后则会从头在开始存，覆盖掉原来的数据
*	printk最后调用call_console_driver实现log_buf数据刷出到指定设备

## 7.3 用户空间console的选择
```C
start_kernel
 -> arch_call_rest_init
  -> rest_init //启动内核init进程
   -> kernel_init
    -> kernel_init_freeable
     -> console_on_rootfs
      -> filp_open("/dev/console", O_RDWR, 0);
```