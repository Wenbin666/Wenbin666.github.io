---
title: Uart简介与TTY驱动
date: 2023-08-07 19:33:51
tags: uart
categories:
 - Peripheral
---
# Uart简介与TTY驱动

# 1. 缩略词说明：
**UART**：Universal Asynchronous Receiver/Transmitter 通用异步收发传输器
**TTY** ：Teletype  电传打字机
**DTR** ：Data Terminal Ready 数据终端就绪
**DSR** ：Data Set Ready 数据配置就绪
**RTS** ：Request To Send 请求发送
**CTS** ：Clear To Send 清除发送

# 2. Uart协议简介
Uart是一种通用串行数据总线，用于异步通信。该总线为双向通信，可以实现全双工传输和接收，串行通信是指利用一条传输线将数据一位位地顺序传送。
特点是通信线路简单。异步通信以一个字符为传输单位，通信中两个字符间的时间间隔多少是不固定的，然而在同一个字符中的两个相邻位间的时间间隔是固定的。
通信基本为三根线：TXD、RXD和GND.
# 2.1 通信格式
![数据通信格式](image.png)
1.	起始位：先发出一个逻辑0的信号，表示传输数据的开始。
2.	数据位：紧接着起始位, 数据位的个数可以是5、6、7、8等，构成一个字符。通常采用ASCII码，从最低位开始传送。
3.	奇偶校验位：数据位加上这一位后，使得"1"的位数应为偶数(偶校验)或奇数(奇校验)，以此来校验资料传送的正确性。常用校验方式：奇校验(ODD)、偶校验(EVEN)，无校验(NONE)。
4.	停止位：它是一个字符数据的结束标志。可以是1位、1.5位、2位的高电平。 停止位不仅仅是表示传输的结束，且提供了通信双方校正时钟同步的机会。
5.	空闲位：处于逻辑“1”状态，表示当前线路上没有数据传送。
![传输示例](image-1.png)

## 2.2 收发FIFO
在UART中引入FIFO缓冲区的概念主要是为了提高数据传输的效率和可靠性：
**处理速度差异**: UART通常用于与其他设备（如微控制器）之间进行串行通信。发送和接收数据的速度可能不同，FIFO可以在数据流传输过程中缓冲数据，有效降低由于速度不匹配导致的数据丢失或溢出。
**减少中断频率**: 如果没有FIFO，每接收到一个字节就需要产生一个中断，这将导致系统频繁地进入中断处理程序，增加了CPU的负担。引入FIFO后，可以在缓冲区中累计多个字节后再触发一次中断，从而减少中断数量，提高系统效率。

## 2.3 UART 中断
常用中断类型如下：
**接收数据中断**：当接收缓冲区中有新的数据字节可供读取时，会产生此中断。
**发送数据中断**：当发送缓冲区空闲且可以接受新的数据字节时，会产生此中断。
**接收缓冲区溢出中断**: 如果接收缓冲区已满且仍继续接收数据，溢出中断会被触发。
**帧错误中断**：如果接收的数据没有正确的帧格式（比如起始位、数据位、停止位不匹配），则会产生帧错误中断，通常用于检测数据传输中的错误。

## 2.4 DMA 模式
配置UART 接收FIFO水位，当水位达到后，UART通过握手线通知DMA，DMA直接参与，把UART输入寄存器的值搬运到内存中，CPU只需要在去检查内存的值，提高了CPU的效率
(可以理解为DMA就是一个搬运工，可以将数据从一个位置搬运到另一个位置)。
使用DMA传输具有以下优点：
**减少CPU负担**：使用DMA模式可以减少CPU的干预，数据在UART和内存之间的传输可以由DMA控制器直接完成。
**提高数据传输效率**：DMA能够在内存和外设之间进行数据传输，而无需CPU的干预，提高了数据传输速度，尤其是在处理大块数据时。
**连续数据传输**：DMA可以在接收或发送数据的同时继续其它操作，减少数据传输的延迟，尤其在高数据速率的应用中尤为重要。

## 2.5 UART 流控
“流”指的是数据流，数据在两个串口之间传输时，可能会出现丢失数据的现象；
若两个串口的处理速度不同，如台式机与单片机之间的通讯，接收端数据缓冲区已满，则此时继续发送来的数据就会丢失。
流控制能解决这个问题，当接收端数据处理不过来时，就发出“不再接收”的信号，发送端就停止发送，直到收到“可以继续发送”的信号再发送数据。

### 2.5.1 软件流控
通过XON/XOFF来实现软件流控制。
当接收端的输入缓冲区内数据量超过设定的高位时，就向数据发送端发出XOFF字符，发送端收到XOFF字符后就立即停止发送数据；
当接收端的输入缓冲区内数据量低于设定的低位时，就向数据发送端发出XON字符，发送端收到XON字符后就立即开始发送数据。
注意事项：若传输的是二进制数据，标志字符也有可能在数据流中出现而引起误操作，这是软件流控制的缺陷，而硬件流控制不会有这个问题。

### 2.5.2 硬件流控
硬件流控制常用的有RTS/CTS流控制和DTR/DSR（数据终端就绪/数据设置就绪）流控制。
硬件流控制必须将相应的电缆线连上，用RTS/CTS（请求发送/清除发送）流控制时，应将通讯两端的RTS、CTS线对应相连。
以PC和modem通信为例，数据终端设备（PC）使用RTS来起始modem的数据流，而modem则用CTS来起动和暂停来自PC的数据流。

# 3. TTY驱动框架
## 3.1 框架说明
![alt text](image-2.png)
TTY整体框架如图所示，具体组成可以分成以下几个方面：
1. 用户层：用户通过系统调用（如 open、read、write、ioctl 等）与TTY驱动进行交互。
2. 字符设备驱动层：TTY驱动作为字符设备的一种，在系统启动时注册，并根据字符设备驱动实现了一组文件操作（file operations），这包括对TTY设备的读写和状态控制。
3. TTY核心层：tty_driver定义了一组函数指针，这些函数指针实现了与 TTY 设备相关的操作。通过这种方式，内核能够以统一的方式与不同的 TTY 设备进行交互。此外tty_driver 可以与 serial_core 层协作，为串行设备提供支持。
4. 串口核心层：serial_core层定义用于串行设备的基本结构和接口，使得多种串口硬件设备可以通过统一接口进行操作。serial_core层采用中断驱动机制来处理数据接收，同时提高系统的响应速度。
5. 硬件层：硬件层涉及具体的串行接口此层负责数据的物理传输，电气特性等。

## 3.2  TTY关键数据结构
### 3.2.1 tty_driver
```C
struct tty_driver {
	int	magic;  //标识结构体的有效性或类型, 用于帮助跟踪错误或验证结构体是否被正确初始化。
	struct kref kref; //引用计数，用于管理此tty_driver结构体的生命周期。它确保在没有任何使用这个驱动的设备时，驱动能够被正确释放和清理。
	struct cdev **cdevs; //指向字符设备结构体的指针数组，用于管理与该 TTY 驱动相关联的所有字符设备
	struct module	*owner; //表示拥有该驱动的内核模块
	const char	*driver_name; //该TTY driver的名称，在软件内部使用
	const char	*name; //该TTY driver所驱动的TTY devices名称，会体现到sysfs以及/dev/
	int	name_base; //TTY 设备名称的基础编号，它在生成设备名称时用作起始点
	int	major; //设备的主设备号，通常用于标识设备的类型
	int	minor_start; //次设备号的起始值
	unsigned int	num; /* number of devices allocated 驱动数量会体现在/dev下面*/
	short	type; //设备的类型，用于描述驱动的特性，可能是控制台、伪终端或其他类型
	short	subtype; //设备的子类型，用于提供更详细的设备信息
	struct ktermios init_termios; //字段用于表示初始化终端的属性，如波特率、字符大小等
	unsigned long	flags; //驱动的标志位，用于特定配置，例如指示驱动是否支持某些特性
	struct proc_dir_entry *proc_entry; //指向proc文件系统条目的指针，允许用户空间访问某些驱动配置或状态信息
	struct tty_driver *other; //指向另一个同类驱动的指针，通常用于管理驱动的备份或冗余
	struct tty_struct **ttys; //指针数组，指向与该驱动相关的 TTY 结构体
	struct tty_port **ports; //指向与该驱动相对应的 TTY 端口结构体
	struct ktermios **termios; //指向终端属性设置的二级指针数组。每个 TTY 设备都有自己的终端属性设置。
	void *driver_state; //指向驱动特定状态数据的指针，供驱动内部使用。
	const struct tty_operations *ops; //tty driver的操作函数集
	struct list_head tty_drivers;  //全局链表头, 挂载所有注册的driver
} __randomize_layout;
```
可以理解为tty devices的具体驱动，driver for TTY devices，比起其他的驱动更像一个设备，因为包含了较多信息，而不是函数。在struct tty_driver中又包含了很多和具体tty操作相关联的数据结构，这些数据结构以tty_为前缀。

### 3.2.2 tty_struct
```C
struct tty_struct {
	int	magic;
	struct kref kref;
	struct device *dev;
	struct tty_driver *driver;
	const struct tty_operations *ops;
	int index; //标识该 TTY 设备在驱动中对应的索引，通常在多设备的情况下用来区分不同的 TTY 实例

	struct ld_semaphore ldisc_sem; //语言描述符（Line discipline）信号量，管理对该 TTY 的访问和同步
	struct tty_ldisc *ldisc; //处理 TTY 数据的输入和输出

	struct mutex atomic_write_lock; //互斥锁，用于保护写操作的原子性，防止在写操作进行时发生竞争条件
	struct mutex legacy_mutex;
	struct mutex throttle_mutex; //互斥锁，管理流控制相关的任务，避免在流控制（throttle）时的状态竞争
	struct rw_semaphore termios_rwsem; //读写信号量，保护终端属性（termios）的读写状态，允许多个读者或单个写者
	struct mutex winsize_mutex; //互斥锁，用于保护窗口大小（winsize）的访问和修改，避免竞争条件
	struct ktermios termios, termios_locked; //termios 结构体实例，存储终端的配置，如波特率、数据位、停止位等，termios_locked 用于存储锁定状态下的配置。
	char name[64]; //终端的名称，通常是在 /dev 目录下可见的设备名称，最大长度为 64 字节
	unsigned long flags;
	int count; //当前引用该 TTY 设备的文件句柄数量，用于管理设备的打开和关闭。
	struct winsize winsize; //表示终端的窗口大小，包括行数和列数等信息

	struct {
		spinlock_t lock;
		bool stopped;
		bool tco_stopped;
		unsigned long unused[0];
	} __aligned(sizeof(unsigned long)) flow; //流控相关

	struct {
		spinlock_t lock;
		struct pid *pgrp;
		struct pid *session;
		unsigned char pktstatus;
		bool packet;
		unsigned long unused[0];
	} __aligned(sizeof(unsigned long)) ctrl; //终端控制相关

	int hw_stopped; //指示硬件停止状态的标记，通常在流控制或硬件状态不正常时使用
	unsigned int receive_room; //接收缓冲区的可用空间，表示可以容纳的字节数
	int flow_change; //用于标识流控制状态的变化，帮助驱动程序管理流控制的逻辑

	struct tty_struct *link; //用于建立 TTY 设备之间的链接或联系
	struct fasync_struct *fasync; //允许用户空间进程监听 TTY 设备的异步事件
	wait_queue_head_t write_wait; //写操作等待队列的头部，允许等待写操作的进程排队
	wait_queue_head_t read_wait; //读操作等待队列的头部，允许等待读操作的进程排队
	struct work_struct hangup_work; //用于管理 TTY 设备挂起的工作结构，通常与挂起事件处理相关
	void *disc_data; //存储描述符特定的状态或数据
	void *driver_data; //存储驱动状态或信息
	spinlock_t files_lock; //自旋锁，用于保护与当前打开的文件句柄相关的状态和信息
	struct list_head tty_files; //自旋锁，用于保护与当前打开的文件句柄相关的状态和信息。

#define N_TTY_BUF_SIZE 4096

	int closing; //该字段用于指示 TTY 设备当前是否正在关闭
	unsigned char *write_buf; //指向字符数组的指针，用于存储待写入 TTY 设备的数据
	int write_cnt; //指示 write_buf 中当前待写入的数据字节数
	struct work_struct SAK_work; //组合键，在需要安全地切换控制台时使用
	struct tty_port *port; //该字段是一个指向 tty_port 结构的指针，表示 TTY 设备的端口（port
} __randomize_layout;
```
tty_struct是TTY设备在TTY core中的内部表示，从TTY driver的角度看，它和文件句柄的功能类似，用于指代某个TTY设备。从TTY core的角度看，它是一个比较复杂的数据结构，保存了TTY设备生命周期中的很多中间变量。

### 3.2.2 tty_port
```C
struct tty_port {
	struct tty_bufhead	buf; //这是一个缓冲区的头部，通常会包含一个用于存储待发送或待接收字节的链表或数组
	struct tty_struct	*tty; // 指向与该端口相关联的 tty_struct 结构体的指针
	struct tty_struct	*itty; //指向相应的 "输入" TTY 设备的指针，可能用于支持集成的 TTY 设备，例如虚拟终端
	const struct tty_port_operations *ops; //定义了一组能对 TTY 端口执行的操作
	const struct tty_port_client_operations *client_ops; //指向与 TTY 端口的客户端操作相关的函数指针
	spinlock_t		lock; // 自旋锁，用于保护对 tty_port 结构的并发访问
	int			blocked_open; // 标志位，用于指示当前端口是否被阻塞打开
	int			count; // 表示当前打开该 TTY 端口的引用计数，即有多少文件句柄引用此端口
	wait_queue_head_t	open_wait; //与打开等待队列相关的头部，允许其他进程在该 TTY 端口被打开时等待
	wait_queue_head_t	delta_msr_wait; //当设备的调制解调器状态发生变化时，该队列中的等待进程可以被唤醒
	unsigned long		flags; //存储 TTY 端口的状态标志位
	unsigned long		iflags; // 输入标志位，表示当前 TTY 端口的输入状态和特性，如数据流控制的条件
	unsigned char		console:1; //指示此端口是否是控制台端口。如果为 1，则表示该端口是控制台
	struct mutex		mutex; // 互斥锁，用于保护对 TTY 端口的访问
	struct mutex		buf_mutex; //互斥锁，专门用于保护缓冲区相关的操作
	unsigned char		*xmit_buf; // 用于存储待传输数据的缓冲区的指针
	DECLARE_KFIFO_PTR(xmit_fifo, unsigned char); //声明一个 FIFO（先进先出）队列，称为 xmit_fifo
	unsigned int		close_delay; //关闭延迟时间（以毫秒为单位）
	unsigned int		closing_wait; // 描述在关闭过程中等待的时间
	int			drain_delay; // 用于管理数据处理过程的延迟时间
	struct kref		kref; // 这是一个引用计数结构，用于在 tty_port 被引用时跟踪其生命周期
	void			*client_data; //指向与客户端相关的用户自定义数据的指针
};
```
tty_port 与tty_struct是可以理解为是相对的关系。
tty_struct是TTY设备的“动态抽象”，保存了TTY设备访问过程中的一些临时信息，这些信息是有生命周期的(从打开TTY设备开始，到关闭TTY设备结束)。
而tty_port是TTY设备固有属性的“静态抽象”，保存了该设备的一些固定不变的属性值，例如具体的操作集、是否是一个控制台设备（console）、打开关闭时是否需要一些延时操作等；
从层次上看，TTY core负责的是逻辑上的抽象，并不关心这些固有属性。这些属性完全可以由具体的TTY driver自行维护。
由于不同TTY设备的属性有很多共性，TTY framework将这些共同的属性抽象出来，保存在struct tty_port数据结构中，同时提供一些通用的操作接口，供具体的TTY driver使用；

### 3.2.3 tty_operations
```C
struct tty_operations {
	struct tty_struct * (*lookup)(struct tty_driver *driver,
			struct file *filp, int idx);
	int  (*install)(struct tty_driver *driver, struct tty_struct *tty);
	void (*remove)(struct tty_driver *driver, struct tty_struct *tty);
	int  (*open)(struct tty_struct * tty, struct file * filp);
	void (*close)(struct tty_struct * tty, struct file * filp);
	void (*shutdown)(struct tty_struct *tty);
	void (*cleanup)(struct tty_struct *tty);
	int  (*write)(struct tty_struct * tty,
		      const unsigned char *buf, int count);
	int  (*put_char)(struct tty_struct *tty, unsigned char ch);
	void (*flush_chars)(struct tty_struct *tty);
	unsigned int (*write_room)(struct tty_struct *tty);
	unsigned int (*chars_in_buffer)(struct tty_struct *tty);
	int  (*ioctl)(struct tty_struct *tty,
		    unsigned int cmd, unsigned long arg);
	long (*compat_ioctl)(struct tty_struct *tty,
			     unsigned int cmd, unsigned long arg);
	void (*set_termios)(struct tty_struct *tty, struct ktermios * old);
	void (*throttle)(struct tty_struct * tty);
	void (*unthrottle)(struct tty_struct * tty);
	void (*stop)(struct tty_struct *tty);
	void (*start)(struct tty_struct *tty);
	void (*hangup)(struct tty_struct *tty);
	int (*break_ctl)(struct tty_struct *tty, int state);
	void (*flush_buffer)(struct tty_struct *tty);
	void (*set_ldisc)(struct tty_struct *tty);
	void (*wait_until_sent)(struct tty_struct *tty, int timeout);
	void (*send_xchar)(struct tty_struct *tty, char ch);
	int (*tiocmget)(struct tty_struct *tty);
	int (*tiocmset)(struct tty_struct *tty,
			unsigned int set, unsigned int clear);
	int (*resize)(struct tty_struct *tty, struct winsize *ws);
	int (*get_icount)(struct tty_struct *tty,
				struct serial_icounter_struct *icount);
	int  (*get_serial)(struct tty_struct *tty, struct serial_struct *p);
	int  (*set_serial)(struct tty_struct *tty, struct serial_struct *p);
	void (*show_fdinfo)(struct tty_struct *tty, struct seq_file *m);
#ifdef CONFIG_CONSOLE_POLL
	int (*poll_init)(struct tty_driver *driver, int line, char *options);
	int (*poll_get_char)(struct tty_driver *driver, int line);
	void (*poll_put_char)(struct tty_driver *driver, int line, char ch);
#endif
	int (*proc_show)(struct seq_file *m, void *driver);
} __randomize_layout;
```
TTY core将和硬件有关的操作，抽象、封装出来，形成名称为struct tty_operations的数据结构，具体的TTY driver不需要关心具体的业务逻辑，只需要根据实际的硬件情况，实现这些操作接口即可。

## 3.3  TTY关键API
### 3.3.1 tty_alloc_driver
```C
#define tty_alloc_driver(lines, flags) \
                __tty_alloc_driver(lines, THIS_MODULE, flags)
extern struct tty_driver *__tty_alloc_driver(unsigned int lines,
                struct module *owner, unsigned long flags);
```
作用：分配一个TTY driver

### 3.3.2 tty_register_driver
```C
int tty_register_driver(struct tty_driver *driver)
```
作用：创建设备节点，并把driver注册到内核中

### 3.3.3 put_tty_driver
```C
extern void put_tty_driver(struct tty_driver *driver);
```
作用：将该tty_driver添加到全局的tty_driver链表中

### 3.3.4 tty_set_operations
```C
void tty_set_operations(struct tty_driver *driver, const struct tty_operations *op);
```
作用：设置该tty_driver和硬件有关的操作集

### 3.3.5 tty_insert_flip_char
```C
static inline int tty_insert_flip_char(struct tty_port *port, unsigned char ch, char flag);
```
作用：向tty_buffer里面插入一个字符

### 3.3.6 tty_insert_flip_string
```C
static inline int tty_insert_flip_string(struct tty_port *port, const unsigned char *chars, size_t size);
```
作用：向tty_buffer里面插字符串

## 3.4 TTY 驱动实现流程
1.	实现TTY设备有关的操作函数集，并保存在一个struct tty_operations变量中。
2.	调用tty_alloc_driver分配一个TTY driver，并根据实际情况，设置driver中的字段（包括步骤1中的struct tty_operations变量）。
3.	调用tty_register_driver将driver注册到kernel。
4.	如果需要动态注册TTY设备，在合适的时机，调用tty_register_device或者tty_register_device_attr，向kernel注册TTY设备。
5.	接收到数据时，调用tty_insert_flip_string或者tty_insert_flip_char将数据交给TTY core；TTY core需要发送数据时，会调用driver提供的回调函数，在那里面访问硬件送出数据。

**说明**： TTY驱动基本不需要自己实现，只需要知道其流程即可。

## 3.5 具体设备发送过程
### 3.5.1 接收数据
```C
 -> uart_driver_read_char
  -> uart_insert_char
   ->tty_insert_flip_char
    -> tty_flip_buffer_push
```
用户空间：
1. 通过read接口获取字符，并激活等待队列
2. 等待下层数据填充
底层：
1.	通过tty_insert_flip_char()，将RX-FIFO中的数据保存在struct tty_buffer中;
2.	通过tty_flip_buffer_push()来唤醒工作队列，即对应 flush_to_ldisc()。
3. 在flush_to_ldisc()中，将数据从struct tty_buffer投递至ldisc层的 ldata.read_buf缓冲区中。

### 3.5.2 发送数据
TTY core需要发送数据时，会调用driver提供的发送函数，在那里面访问硬件送出数据

# 4. 总结

TTY 驱动是 Linux 内核中管理和操作终端设备的重要组成部分，为用户空间与终端的通信提供了统一接口和高级抽象。
而 UART 则负责底层的串行通信实现，为 TTY 驱动提供必要的数据传输能力。二者通过协调工作，实现从硬件到用户空间的完整数据流动。