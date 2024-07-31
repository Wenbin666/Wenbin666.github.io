---
title: Linux spi驱动分析
date: 2022-06-29 19:39:34
tags: spi
categories:
 - Peripheral
---
# 1. 简介
Spi（Serial Peripheral Interface）即串行外设接口，是一种高速的、全双工、同步的通信总线，由Motorola公司在20世纪80年代中期开发，后发展成了行业规范。spi广泛应用于嵌入式系统中的短距离通信，特别是在EEPROM、FLASH、ADC、DAC等芯片，以及数字信号处理器和数字信号解码器之间的通信中。
![SPI通信主从链接拓扑](image.png)
## 1.1 特点
* 标准四线通信：
    + SCK (Serial Clock)：串行时钟信号，由Master产生，用于同步数据传输
    + MOSI (Master Output, Slave Input)： Master输出/Slave输入引脚，用于Master向Slave发送数据
    + MISO（Master Input, Slave Output）：Master输入/Slave输出引脚，用于Slave向Master发送数据
    + CS/SS（Chip Select/Slave Select）：Slave片选信号，由Master控制，用于选择特定Slave进行通信
* 主多从架构：一个主设备和多个从设备，通过片选线决定与哪个Slave设备通信(**拉低片选**)
* 全双工通信：允许Master和Slave同时发送和接收数据，双向通信，提高了通信效率
* 同步通信：通过时钟信号来同步数据传输，确保数据的准确性和稳定性
* 通信速率：受Master和Slave规格性能、时钟频率等影响，通常为20Mbps到50Mbps，最高可达几百Mbps

在某些情况下，也可以用3根线的spi或者2根线的spi进行通信：
1. Master只给Slave发送命令，Slave不需要回复数据时，MISO线可以去掉；
2. 在Master只读取Slave的数据，不需要给Slave发送指令的时候，MOSI线可以去掉；
3. 当只有一个Master一个Slave的时候，Slave的片选可以固定为有效电平，即使能状态，CS/SS线可以去掉；

## 1.2 通信时序
![](image-2.png)
spi通信时序比较简洁明了可以总结为以下几点：
1) CS/SS信号低电平有效，当Master要与某外设通信的时，需要将该外设上的CS/SS线拉低，类似与I2C中的Start信号
2) 在每个时钟周期内，主设备通过MOSI线向从设备发送一位数据，同时从设备通过MISO线向主设备发送一位数据
3) 注意spi从设备支持的spi总线最高时钟频率（决定了SCK的频率）

## 1.3 通信模式
学习通信模式之前，需要先搞懂两个概念：CPOL和CPHA
**CPOL**: Clock Polarity 决定时钟信号在空闲状态时的电平(高或低)。
**CPHA**: Clock Phase 决定数据是在时钟信号的哪个边沿（上升沿或下降沿）被采样。
解释：
Master和Slave要交换数据，就牵涉到一个问题：即Master在什么时刻输出数据到MOSI上而Slave在什么时刻采样这个数据，或者Slave在什么时刻输出数据到MISO上而Master什么时刻采样这个数据。
同步通信的一个特点是**所有数据的变化和采样都是伴随着时钟沿进行的**，也就是说数据总是在时钟的边沿附近变化或被采样。而一个时钟周期包含了一个上升沿和一个下降沿，只是这两个沿的先后并无规定。又因为数据从产生的时刻到它的稳定是需要一定时间的，那么，如果Master在上升沿输出数据到MOSI上，Slave就只能在下降沿去采样这个数据了。反之如果一方在下降沿输出数据，那么另一方就必须在上升沿采样这个数据。

背景：
spi协议最初在1987年左右开发，摩托罗拉公司旨在提供一种简单而有效的串行通信方式，并没有固定Clock Polarity（CPOL）和Clock Phase（CPHA）的值，而这两个值由各个厂商在产品设计时自行决定。这就导致了目前存在的四种工作模式(排列组合得到)：
![](image-1.png)
* 如果CPOL=0，串行同步时钟的空闲状态为低电平；
* 如果CPOL=1，串行同步时 钟的空闲状态为高电平。
* 如果CPHA=0，在串行同步时钟的第一个跳变沿（上升或下降）数据被采样；
* 如果CPHA=1，在串行同步时钟的第二个跳变沿（上升或下降）数据被采样。

## 1.4 通信优缺点
### 1.4.1 优点
**无需寻址**：传输无需寻址，简单高效
**高速数据传输**：全双工的工作方式，数据可以同时在两个方向上传输，传输效率较高
**灵活性**：支持多个从设备的连接，通过过调整时钟频率，还可以适应不同速度的外围设备

### 1.4.2 缺点
**无内置的流控和应答机制**：不提供流控制机制来确保数据传输的流畅性，也没有应答机制来确认从设备是否已正确接收数据。这要求应用层实现额外的错误检测和重传机制来确保数据的可靠传输。
**占用主机口线较多**：每个从设备都需要一根独立的片选线来控制其通信
**距离限制**：适用于短距离通信。在较长距离的通信中，由于信号衰减和干扰等问题，spi通信的可靠性和稳定性可能会受到影响。
**依赖于时钟同步**：依赖于时钟信号的同步来确保数据的正确传输。如果时钟信号不稳定或受到干扰，可能会导致数据传输错误。

# 2. Linux spi框架
## 2.1 架构分析
linux系统下的spi驱动程序从逻辑上可以分为3个部分：
![](image-3.png)
**spi core**：Linux内核用来维护和管理spi的核心部分
**spi bus drivers**(主机控制器驱动层)：spi Master针对不同类型的spi控制器硬件，实现spi总线的硬件访问操作。
**spi device driver**：对应于spi Slave设备端的驱动程序，spi driver的作用是将spi设备挂接到spi总线上；

Linux中的主从模式的总线子系统采用的是同一种分离思想，其分离的具体策略大同小异，同样分为设备驱动层、核心层、总线驱动层。
具体的分离策略分析可参考[Linuxi2c驱动分析](https://wenbin666.github.io/2023/05/09/i2c/) 中内核对I2C子系统框架的说明。
结合数据结构进行对比，其中原委更加清晰明了：
| 总线类型             | I2C                   | SPI                   |
|------------------|-----------------------|-----------------------|
| master controler | struct i2c_adapter    | struct spi_master     |
| 控制器操作方法          | struct i2c_algorithm  | struct spi_bitbang    |
| slave设备          | struct i2c_client     | struct spi_device     |
| slave设备板卡信息      | struct i2c_board_info | struct spi_board_info |
| slave设备驱动        | struct i2c_driver     | struct spi_driver     |
| 一次完整的数据包         | struct i2c_msg        | struct spi_transfer   |
| 多个完整数据包          |            -          | struct spi_message    |

## 2.2 文件结构说明
```C
├── spi.c
├── spidev.c
├── spi-gpio.c
├── spi-bitbang.c
├── spi-imx.c
├── spi-atmel.c
```
**spi.c**：SPI子系统的核心实现文件，包含了SPI总线的初始化、注册以及相关的API接口
**spidev.c**：用于在用户空间和内核空间之间提供SPI通信的能力。
**spi-gpio.c**：提供了使用GPIO引脚模拟SPI总线的框架，允许在没有硬件SPI控制器的情况下实现SPI通信
**spi-bitbang.c**：通用框架，用于在没有硬件SPI控制器或需要灵活控制SPI时序和配置的系统中模拟SPI总线的通信
**控制器驱动文件**：如spi-altera.c、spi-atmel.c等，这些文件由不同芯片厂商提供，用于适配各自的SPI控制器硬件

## 2.3 关键数据结构(v5.10.17)
### 2.3.1 spi_master
```C
#define spi_master spi_controller
struct spi_controller {
	struct device	dev; //设备相关
	struct list_head list; // 双向链表头，用于将SPI控制器链接到全局的SPI控制器列表中
	s16			bus_num;   // 总线的编号，唯一标识一个 SPI 控制器
	u16			num_chipselect; //片选信号的数量
	u16			dma_alignment; // dma传输的对齐要求
	u32			mode_bits; //模式类型
	u32			buswidth_override_bits; //允许mask的SPI总线宽度的位掩码
	u32			bits_per_word_mask; //每个字的比特数掩码，定义了传输数据的位宽
	u32			min_speed_hz;
	u32			max_speed_hz;
	u16			flags;
	bool			devm_allocated；//标记控制器是否使用 devm_ 前缀的函数分配资源
	bool			slave; //标识控制器是否作为 SPI 从设备
	size_t (*max_transfer_size)(struct spi_device *spi); //设置最大传输大小
	size_t (*max_message_size)(struct spi_device *spi);
	struct mutex		io_mutex;
	struct mutex		add_lock;
	spinlock_t		bus_lock_spinlock;
	struct mutex		bus_lock_mutex;
	bool			bus_lock_flag;
	int			(*setup)(struct spi_device *spi); //设置SPI控制器和工作方式、clock等
	int (*set_cs_timing)(struct spi_device *spi); //设置时序
	int			(*transfer)(struct spi_device *spi,
						struct spi_message *mesg); //传输函数
	void			(*cleanup)(struct spi_device *spi);
	bool			(*can_dma)(struct spi_controller *ctlr,
					   struct spi_device *spi,
					   struct spi_transfer *xfer);
	struct device *dma_map_dev;
	bool				queued;
	struct kthread_worker		*kworker;
	struct kthread_work		pump_messages;
	spinlock_t			queue_lock;
	struct list_head		queue;
	struct spi_message		*cur_msg; //传输的message
	bool				idling;
	bool				busy;
	bool				running;
	bool				rt;
	bool				auto_runtime_pm;
	bool                cur_msg_prepared;
	bool				cur_msg_mapped;
	char				last_cs;
	bool				last_cs_mode_high;
	bool                            fallback;
	struct completion               xfer_completion;
	size_t				max_dma_len;

	int (*prepare_transfer_hardware)(struct spi_controller *ctlr); //封装的函数指针
	int (*transfer_one_message)(struct spi_controller *ctlr,
				    struct spi_message *mesg);//传输一个完整的 SPI 消息
	int (*unprepare_transfer_hardware)(struct spi_controller *ctlr);
	int (*prepare_message)(struct spi_controller *ctlr,
			       struct spi_message *message);
	int (*unprepare_message)(struct spi_controller *ctlr,
				 struct spi_message *message);
	int (*slave_abort)(struct spi_controller *ctlr);
	void (*set_cs)(struct spi_device *spi, bool enable);//用于控制 SPI 设备的片选
	int (*transfer_one)(struct spi_controller *ctlr, struct spi_device *spi,
			    struct spi_transfer *transfer);
	void (*handle_err)(struct spi_controller *ctlr,
			   struct spi_message *message);
	const struct spi_controller_mem_ops *mem_ops;
	const struct spi_controller_mem_caps *mem_caps;
	struct gpio_desc	**cs_gpiods;
	bool			use_gpio_descriptors;
	s8			unused_native_cs;
	s8			max_native_cs;
	struct spi_statistics	statistics;
	struct dma_chan		*dma_tx;
	struct dma_chan		*dma_rx;
	void			*dummy_rx;
	void			*dummy_tx;

	int (*fw_translate_cs)(struct spi_controller *ctlr, unsigned cs);
	bool			ptp_sts_supported;
	unsigned long		irq_flags;
};
```
spi_master代表一个spi控制器。它封装了SPI通信的复杂性和底层细节，提供了简单、高效的接口，使得用户能够轻松地实现与各种外设的SPI通信。

### 2.3.2 spi_device
```C
struct spi_device {
	struct device		dev; 
	struct spi_controller	*controller; //指向控制该SPI设备的SPI控制器
	struct spi_controller	*master;	/* 指向控制该SPI设备的SPI控制器, 兼容考虑 */
	u32			max_speed_hz; // SPI设备支持的最大通信速率
	u8			chip_select;  // 片选
	u8			bits_per_word; // 定义每次通信中传输的数据位数。常见的值有8位和16位
	bool			rt;
	u32			mode; // 模式类型
	int			irq;
	void			*controller_state;
	void			*controller_data;
	char			modalias[SPI_NAME_SIZE];
	const char		*driver_override;
	struct gpio_desc	*cs_gpiod;	/* chip select gpio desc */
	struct spi_delay	word_delay; /* inter-word delay */
	/* CS delays */
	struct spi_delay	cs_setup; // 片选延迟
	struct spi_delay	cs_hold;
	struct spi_delay	cs_inactive;

	/* the statistics */
	struct spi_statistics	statistics;
};
```
spi_device用户与描述一个spi设备和其属性，由master controller driver注册完成后扫描BSP中注册设备产生的设备链表并向spi_bus注册产生。

### 2.3.3 spi_driver
```C
struct spi_driver {
	const struct spi_device_id *id_table;
	int			(*probe)(struct spi_device *spi);
	void			(*remove)(struct spi_device *spi);
	void			(*shutdown)(struct spi_device *spi);
	struct device_driver	driver;
};
```
用于定义和描述SPI设备的驱动程序。这个结构体是SPI设备驱动与Linux内核SPI子系统之间交互的桥梁，它封装了驱动程序的必要信息，使得内核能够识别、加载、绑定和管理SPI设备驱动。

### 2.3.4 spi_board_info
```C
struct spi_board_info {
	char		modalias[SPI_NAME_SIZE]; //字符数组，用于存储设备的模块别名
	const void	*platform_data; // 用于传递额外的配置信息给驱动程序
	const struct software_node *swnode; //允许SPI子系统直接访问与设备相关联的设备树节点，从而获取更详细的配置信息
	void		*controller_data; //用于在SPI控制器和设备之间传递额外的数据或状态信息
	int		irq;
	u32		max_speed_hz;
	u16		bus_num;
	u16		chip_select;
	u32		mode;
};
```
该结构体记录着SPI外设使用的主机控制器序号、片选序号、数据比特率、SPI传输式(即CPOL、CPHA)等板级信息。

### 2.3.5 spi_message
```C
struct spi_message {
	struct list_head	transfers; //定义了SPI传输中的一个或多个数据帧的详细参数，如传输方向、数据长度、速度等
	struct spi_device	*spi; //表示这次SPI消息传输的目标设备
	unsigned		is_dma_mapped:1; //指示这次SPI消息传输是否使用了DMA映射

	/* completion is reported through a callback */
	void			(*complete)(void *context);
	void			*context; // 调用complete回调函数时传递用户自定义的上下文信息
	unsigned		frame_length;
	unsigned		actual_length; //用于记录SPI消息传输过程中实际传输的帧数或字节数
	int			status; //记录SPI消息传输的状态
	struct list_head	queue;
	void			*state;
	/* list of spi_res reources when the spi message is processed */
	struct list_head        resources;
};
```
代表了一次完整的SPI传输任务。这个结构体封装了与一次SPI消息传输相关的所有信息，包括传输的数据、目标SPI设备、传输状态、回调函数等。

### 2.3.6 spi_transfer
```C
struct spi_transfer {
	const void	*tx_buf; //发送缓冲区指针，用于发送数据给SPI从设备
	void		*rx_buf; //接收缓冲区指针，用于从SPI设备收数据
	unsigned	len;  // 传输的数据长度（以字节为单位）
	dma_addr_t	tx_dma;  //dma 相关 发送地址 
	dma_addr_t	rx_dma;  //dma 相关 接收地址 
	struct sg_table tx_sg;
	struct sg_table rx_sg;
	unsigned	dummy_data:1; //是否发送虚拟数据。在某些SPI设备需要特定的时钟脉冲来同步或初始化时非常有用
	unsigned	cs_change:1; //用于指示在传输完成后是否需要改变片选（CS）信号的状态
	unsigned	tx_nbits:3; //定义发送数据的位数。支持单比特、双比特和四比特传输模式
	unsigned	rx_nbits:3; //定义接收数据的位数。
#define	SPI_NBITS_SINGLE	0x01 /* 1bit transfer */
#define	SPI_NBITS_DUAL		0x02 /* 2bits transfer */
#define	SPI_NBITS_QUAD		0x04 /* 4bits transfer */
	u8		bits_per_word; //配置SPI通信的位宽
	struct spi_delay	delay;
	struct spi_delay	cs_change_delay;
	struct spi_delay	word_delay;
	u32		speed_hz;
	u32		effective_speed_hz;
	unsigned int	ptp_sts_word_pre;
	unsigned int	ptp_sts_word_post;
	struct ptp_system_timestamp *ptp_sts;
	bool		timestamped;
	struct list_head transfer_list; //用于spi_transfer结构体链接到更大的spi_message结构体中的transfers链表中
#define SPI_TRANS_FAIL_NO_START	BIT(0)
	u16		error;
};
```
这个结构体包含了传输过程中所需的所有参数和状态信息，以便SPI控制器能够正确地执行数据传输。

### 2.3.7 数据结构的关系
![](image-4.png)
**spi_master 与 spi_device**：spi_master代表SPI主控制器，管理SPI总线上的通信；spi_device代表SPI总线上的一个从设备，是通信的目标。它们之间的关系是主从关系，即spi_master控制并与spi_device进行通信。
**spi_board_info 与 spi_device**：spi_board_info提供了spi_device所需的配置信息，是创建spi_device的基础。在SPI主控制器初始化时，会根据spi_board_info中的信息创建对应的spi_device实例。
**spi_transfer 与 spi_message**：spi_transfer是SPI通信的基本单位，而spi_message则用于组织多个spi_transfer以实现更复杂的通信逻辑。它们之间的关系是组成关系，即多个spi_transfer可以组成一个spi_message。

# 3. 关键API
## 3.1 总线相关API
### 3.1.1 spi_alloc_master
```C
static inline struct spi_controller *spi_alloc_master(struct device *host,
						      unsigned int size);
```
分配并初始化一个spi_master(spi_controller)结构体。
host：SPI主控制器所在的设备结构体。
size：为私有数据预留的空间大小。
返回值：指向分配并初始化的spi_master结构体的指针，失败时返回NULL。

### 3.1.2 spi_register_master
```C
#define spi_register_master(_ctlr)	spi_register_controller(_ctlr)
int spi_register_controller(struct spi_controller *ctlr);
```
功能：向SPI子系统注册一个SPI控制器。
这个函数是SPI框架提供的一个核心接口，它允许SPI控制器硬件的驱动程序将其控制器实例注册到内核的SPI子系统中，从而使得其他SPI设备驱动程序能够找到并使用这个控制器来与SPI设备进行通信。
*ctlr：这是一个指向spi_controller结构体的指针
返回值：成功时，该函数返回0。如果注册失败，则返回负的错误码。

### 3.1.3 spi_unregister_master
```C
#define spi_unregister_master(_ctlr)	spi_unregister_controller(_ctlr)
void spi_unregister_controller(struct spi_controller *ctlr)
```
当SPI控制器驱动程序在不再需要SPI控制器或者当SPI控制器硬件被移除时调用的

## 3.2 驱动相关API
### 3.2.1 spi_register_driver
```C
#define spi_register_driver(driver) __spi_register_driver(THIS_MODULE, driver)
int __spi_register_driver(struct module *owner, struct spi_driver *sdrv);
```
将SPI设备驱动程序注册到SPI子系统中，如果注册成功，函数返回0。如果注册失败，则返回负的错误码。

### 3.2.2 spi_unregister_driver
```C
static inline void spi_unregister_driver(struct spi_driver *sdrv)；
```
注销之前注册的SPI设备驱动，sdrv为要注销的spi_driver结构体指针。

## 3.3 传输相关API
### 3.3.1 spi_message_init
```C
static inline void spi_message_init(struct spi_message *m);
```
初始化一个spi消息队列

### 3.3.2 spi_message_init
```C
spi_message_add_tail(struct spi_transfer *t, struct spi_message *m);
```
将spi_transfer t加入到消息 m的的链表中

### 3.3.3 spi_sync
```C
int spi_sync(struct spi_device *spi, struct spi_message *message);
```
以阻塞的方式与SPI设备进行通信，直到整个消息队列被完全处理完毕。
*spi：指向spi_device结构体的指针，这个参数指定了要与哪个SPI设备进行通信。
*message：要传输的消息信息
返回值：成功时，返回0。如果在传输过程中发生错误，则返回负的错误码。

### 3.3.4 spi_async
```C
int spi_async(struct spi_device *spi, struct spi_message *message);
```
用于异步地发送一个SPI消息队列，非阻塞执行，消息发送完毕可以调用相应的回调函数。


# 4.驱动关键流程
## 4.1 注册spi控制器
注册spi控制器到内核分为两个阶段：
1) 通过spi_alloc_master,分配一个spi_master的空间
2) 通过spi_register_master将第一阶段分配的spi_master注册到内核中

## 4.2 传输流程
1. 定义一个spi_message结构；
2. 用spi_message_init函数初始化spi_message；
3. 定义一个或数个spi_transfer结构，初始化并为数据准备缓冲区并赋值给spi_transfer相应的字段；
4. 通过spi_message_init函数把这些spi_transfer挂在spi_message结构下；
5. 如果使用同步方式，调用spi_sync，如果使用异步方式，调用spi_async发送消息。

## 4.3 注销spi控制器
通过spi_unregister_master将spi主控制器从内核中取消注册

# 5. 总结
本文分析了spi规范、linux驱动中spi相关的关键结构体和api，以及描述了api传输的简单流程，仅做了解，后续有需求再补充完善。



