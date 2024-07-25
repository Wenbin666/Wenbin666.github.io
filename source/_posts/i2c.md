---
title: Linux I2C 驱动分析
date: 2023-05-09 10:31:25
tags: i2c
categories:
 - Peripheral
---
# Linux I2C 驱动分析

# 1. 简介
I2C(Inter－Integrated Circuit)是一种通用的总线协议。它是由Philips(飞利浦)，现NXP(恩智浦)半导体开发的一种简单的双向两线制总线协议标准。
* I2C其实指代的是I2C bus，也就是内部集成电路总线，一主多从架构
* 两线制：开漏输出(只有下拉能力)，串行数据线(SDA, Serial Data line)和时钟线（SCL，Serial CLock line），均需外接上拉电阻，典型电压为1.8V、3.3V等
![](image.png)

# 2. 特性
* 半双工通信
* 7bit或10bit地址传输模式
* 常用速率400Kbit/s，支持速率范围从100Kbits到3.4Mbit/s
* 支持多个slave设备，slave设备是7bit地址模式时，理论上可以挂2^7=128个设备，但是除去保留地址，只支持112个slave设备
* 协议规定总线电容不超过400pf，实际使用中一条总线上尽量不要超过8个slave设备。
* 支持clock stretch, 从设备可以通过拉低SCL线来进行clock stretching的操作，以暂停通信过程，直到从设备准备好继续传输

# 3. 角色与术语
## 3.1 设备角色
**Transmitter**：向总线发送数据的设备。发送方既可以是向总线发起数据传输的设备Master，也可以是响应Master请求向总线发送数据的设备Slave。
**Receiver**：从总线接收数据的设备。接收方可以是根据自己的请求接收数据的设备Master，也可以是响应Master的请求slave。
**Master**：初始化传输（发出START）、生成时钟（SCL）信号并终止传输（STOP）的组件。
**Slave**：主设备寻址的设备。

## 3.2 总线传输术语
### 3.2.1 START(RESTART)
数据搬移以START或RESTART条件开始，此时SDA数据线从高到低变化，而SCL时钟线保持高。
当这种情况发生时，总线处于busy状态。
**注意**：Restart的出现本质是为了修改读写位，比如之前的读，后续需要写。

### 3.2.2 STOP
数据搬移由STOP条件终止。
当SDA数据线从低电平状态变为高电平状态，而SCL时钟线保持高电平时。数据传输终止时，总线处于空闲状态。
![Start与Stop信号](image-1.png)
**说明**：在SCL为高时，SDA上升沿或下降沿代表启动或停止，SCL为低的时候，SDA才能跳变，代表数据传输。

### 3.2.3 ACK与NACK
SCL的第9个时钟周期用于ACK和NACK确认与否。
Master在此时钟脉冲期间，会释放SDA线，Slave可以将SDA线拉低，并在时钟脉冲的高电平期间保持稳定的低电平,为ACK，否则为NACK
![](image-2.png)

# 4. 总线协议
## 4.1 数据读写
### 4.1.1 写数据（主发从收）
![](image-3.png)
所有数据都以字节格式传输，每次数据搬移传输的字节数没有限制，基本传输流程如下：
* Master SCL保持高电平，SDA下拉，发送START信号
* Mater发送Slave设备地址，确认要通信的目标
* 接着是读写位，此处为0，代表写数据
* Slave 拉低SDA，发送ACK应答信号
* Master发送要写的数据值
* Slave发送ACK应答信号
* Master发送要写的数据值
* Slave发送ACK应答信号
* 数据发送完毕，Master发送Stop信号，本次传输完毕

### 4.1.2 读数据读（主收从发，此处包含了RESTART,为常用使用场景）
![](image-4.png)
* Master SCL保持高电平，SDA下拉，发送START信号
* Mater发送Slave设备地址，确认要通信的目标
* 接着是读写位，此处为0，代表写数据(需要告诉slve要读的是哪个地址，所以需要先写)
* Slave 拉低SDA，发送ACK应答信号
* Master发送要写的数据值（代表预期读的寄存器值）
* Slave发送ACK应答信号
* Master SCL保持高电平，SDA下拉，发送**RESTART**信号
* Mater发送Slave设备地址，确认要通信的目标(slave地址 + 写地址偏移)
* 接着是读写位，此处为1，代表读数据
* Slave发送数据（Master要读的寄存器值）
* Master发送NACK应答信号，代表不继续读
* Master发送Stop信号，本次传输完毕

## 4.2 寻址模式与地址保留
### 4.2.1 七位寻址
在7位地址格式中，Master首先发送数据的前七位（bit[7:1]）设置Slave地址，接着(bit[0])为R/W位，当（R/W）设置为0时，Master写Slave。当设置为1时，Master读Slave。
![](image-5.png)
### 4.2.2 十位寻址
在10位寻址过程中，传输两个字节来设置10位地址:
第一个字节的前五位（bit[7:3]为"11110"）通知Slave，这是一个10位寻址模式，然后是接下来的两位（bit[2:1]），为Slave地址位的[9:8]，接着是R/W位，传输的第二个字节表示Slave地址的bit[7:0]。
![](image-6.png)
### 4.2.3 地址保留
I2C规范中保留了一些地址，用于一些特殊用途，进一步解释了为什么7bit地址到不了128个。
| Address  | R/W bits | Description                                      |
|----------|----------|--------------------------------------------------|
| 0000 000 | 0        | General call：允许一个主设备同时与所有连接到总线上的从设备进行通信          |
| 0000 000 | 1        | 起始信号                                             |
| 0000 001 | X        | CBUS 地址                                          |
| 0000 010 | X        | 暂未使用，保留地址                                        |
| 0000 011 | X        | 暂未使用，保留地址                                        |
| 0000 1XX | X        | 高速模式传输code,(3.4M高速波形与其它速度模式完全不同)                 |
| 1111 1XX | X        | 暂未使用，保留地址                                        |
| 1111 0XX | X        | 10位从机地址                                          |
| 0001 000 | X        | SMBus Host：用于与 SMBus（System Management Bus）相关的通信 |
| 0001 000 | X        | SMBus Alert Response Address                     |
| 1100 001 | X        | SMBus Device Default Address                     |

**注意**：协议不限制使用这些保留地址。但如果使用这些保留地址地址，可能会遇到与其他I2C组件不兼容的问题，建议不使用。

## 4.3 Clock Stretching
通常情况下，由I2C Master控制时钟速度并在线路上提供 SCL。Slave可以“延长”或持续hold时钟来暂停Master的数据或命令传输。
Clock Stretching主要用于在从设备忙乱时保持和主设备通信关系。Slave可以使用Clock Stretching将主设备推入等待状态。当从设备需要更长的时间来管理数据时，它可能会执行时钟延长。
Clock Stretching通常发生在从设备确认接收到一个字节的数据之后。
![](image-7.png)

## 4.4 多主机仲裁（线与关系仲裁）
当两个或多个Master同时试图在总线上发起传输时，会发生多Master仲裁和同步SCL时钟，来选出最后由哪个Master获得真正的传输权力。
仲裁发生在SDA信号线上，当SCL线为1时（需要发送START信号，也就是SDA拉低）。
Master当发送一个1时，当其他的Master发送一个0时(发送1的就输了，没抢到Orz！)，失去仲裁权并关闭其数据输出。
如果两个主机都寻址同一个从设备，仲裁可能会进入数据阶段。直到它确实失去仲裁权已经给到另一个Master，它就会停止生成SCL;
![两个Master仲裁过程](image-8.png)

# 5. Linux I2C 体系架构
# 5.1 体系简介
![I2C体系架构图](image-10.png)
I2C体系架构如图所示，以用户控制一个具体的I2C设备为例：
1) 用户App通过read/write等接口访问具体的dev设备访问相应i2c，如/dev/i2c-1,
2) 经过i2c_dev进行抽象，变成数据的发送与传输，并通过i2c-core与algorithm变成msg的发送与接收。
3) i2c 适配器将msg进行解析，分成更底层的对具体I2C设备的寄存器控制

如果从分层的角度看，Linux的I2C体系结构分为i2c core、 i2c bus driver 和 i2c device driver三层：
**i2c core**:提供了i2c总线驱动和设备驱动的注册、注销方法、通信方法(即Algorithm)和上层的与具体适配器无关的代码以及探测设备、检测设备地址的上层代码等（kernel原生封装实现）。
**i2c bus driver**：对I2C硬件体系结构中Adapter端的实现, 主要包含I2C适配器相关数据结构i2c_adapter、i2c_algorithm和控制i2c适配器产生通信信号的函数（IP厂家提供）。
**i2c device driver**：i2C设备驱动，是对12C硬件体系结构中设备端的实现：I2C设备驱动主要包含i2c_driver和i2c_client等。

## 5.2 I2C 文件架构与说明
```C
├── algos
│   ├── i2c-algo-bit.c
│   ├── i2c-algo-pca.c
├── busses
│   ├── i2c-tegra.c
│   ├── i2c-ormap.c
│   ├── i2c-s3c2410.c
├── muxes
│   ├── i2c-mux-gpio.c
│   ├── i2c-mux-gpmux.c
│   ├── i2c-mux-ltc4306.c
├── i2c-core-base.c
├── i2c-core.h
├── i2c-core-of.c
├── i2c-core-slave.c
├── i2c-core-smbus.c
├── i2c-dev.c
├── i2c-mux.c
```
文件说明：
**i2c-core-base.c**：实现I2C core的功能，它负责处理 I2C 设备的注册、管理和与用户空间的交互。
**i2c-dev.c**: 实现I2C适配器设备文件的功能,可以通过文件操作接口open()、write()、read()、ioctl()和close()等来访问设备。
**busses**: 文件夹、包含一些主流I2C 控制器的驱动, 如i2c-tegra.c、i2c-ormap.c、i2c-versatile.c、i2c-s3c2410.c等。
**algos**: 实现I2C总线适配器的通信方法。
此外：此外内核中的i2c.h头文件对i2c_adapter、i2c_algorithm、i2c_driver和i2c_client这4个数据结构进行定义。

## 5.3 I2C数据结构
### 5.3.1 i2c_adapter
```C
struct i2c_adapter {
	struct module *owner;
	unsigned int class;		  /* classes to allow probing for */
	const struct i2c_algorithm *algo; /* the algorithm to access the bus */
	void *algo_data;
	/* data fields that are valid for all devices	*/
	const struct i2c_lock_operations *lock_ops;
	struct rt_mutex bus_lock;
	struct rt_mutex mux_lock;
	int timeout;			/* in jiffies */
	int retries;
	struct device dev;		/* the adapter device */
	unsigned long locked_flags;	/* owned by the I2C core */
	int nr;
	char name[48];
	struct completion dev_released;
	struct mutex userspace_clients_lock;
	struct list_head userspace_clients;
	struct i2c_bus_recovery_info *bus_recovery_info;
	const struct i2c_adapter_quirks *quirks;
	struct irq_domain *host_notify_domain;
	struct regulator *bus_regulator;
};
```
i2c_adapter对应于物理上的一个适配器及访问它所需的访问算法的结构。

### 5.3.2 i2c_algorithm
```C
struct i2c_algorithm {
	int (*master_xfer)(struct i2c_adapter *adap, struct i2c_msg *msgs,
			   int num);
	int (*master_xfer_atomic)(struct i2c_adapter *adap,
				   struct i2c_msg *msgs, int num);
	int (*smbus_xfer)(struct i2c_adapter *adap, u16 addr,
			  unsigned short flags, char read_write,
			  u8 command, int size, union i2c_smbus_data *data);
	int (*smbus_xfer_atomic)(struct i2c_adapter *adap, u16 addr,
				 unsigned short flags, char read_write,
				 u8 command, int size, union i2c_smbus_data *data);

	/* To determine what the adapter supports */
	u32 (*functionality)(struct i2c_adapter *adap);
};
```
i2c_algorithm对应一套通信方法。
其中**master_xfer**对应为I2C传输函数指针, I2C Master驱动的大部分工作都在这个函数之中。
#### 5.3.2.1 i2c_msg
```C
struct i2c_msg {
	__u16 addr;
	__u16 flags;
#define I2C_M_RD		0x0001	/* guaranteed to be 0x0001! */
#define I2C_M_TEN		0x0010	/* use only if I2C_FUNC_10BIT_ADDR */
#define I2C_M_DMA_SAFE		0x0200	/* use only in kernel space */
#define I2C_M_RECV_LEN		0x0400	/* use only if I2C_FUNC_SMBUS_READ_BLOCK_DATA */
#define I2C_M_NO_RD_ACK		0x0800	/* use only if I2C_FUNC_PROTOCOL_MANGLING */
#define I2C_M_IGNORE_NAK	0x1000	/* use only if I2C_FUNC_PROTOCOL_MANGLING */
#define I2C_M_REV_DIR_ADDR	0x2000	/* use only if I2C_FUNC_PROTOCOL_MANGLING */
#define I2C_M_NOSTART		0x4000	/* use only if I2C_FUNC_NOSTART */
#define I2C_M_STOP		0x8000	/* use only if I2C_FUNC_PROTOCOL_MANGLING */
	__u16 len;
	__u8 *buf;
};
```
存储I2C传输的msg信息：包含传输地址、方向、缓冲区、缓冲区长度等信息。(include/uapi/linux/i2c.h, 在uapi下，用户空间也会使用这个结构体)

### 5.3.3 i2c_driver
```C
struct i2c_driver {
	unsigned int class;
	int (*probe)(struct i2c_client *client, const struct i2c_device_id *id);
	int (*remove)(struct i2c_client *client);

	int (*probe_new)(struct i2c_client *client);
	void (*shutdown)(struct i2c_client *client);

	void (*alert)(struct i2c_client *client, enum i2c_alert_protocol protocol,
		      unsigned int data);

	int (*command)(struct i2c_client *client, unsigned int cmd, void *arg);

	struct device_driver driver;
	const struct i2c_device_id *id_table; //所支持的ID表格

	int (*detect)(struct i2c_client *client, struct i2c_board_info *info);
	const unsigned short *address_list;
	struct list_head clients;
	u32 flags;
};
```
表示I2C设备driver相关，所以基本都是函数实现，包括标准的probe、remove等。
### 5.3.4 i2c_client
```C
struct i2c_client {
	unsigned short flags;		/* div., see below		*/
#define I2C_CLIENT_PEC		0x04	/* Use Packet Error Checking */
#define I2C_CLIENT_TEN		0x10	/* we have a ten bit chip address */
#define I2C_CLIENT_SLAVE	0x20	/* we are the slave */
#define I2C_CLIENT_HOST_NOTIFY	0x40	/* We want to use I2C host notify */
#define I2C_CLIENT_WAKE		0x80	/* for board_info; true iff can wake */
#define I2C_CLIENT_SCCB		0x9000	/* Use Omnivision SCCB protocol */
					/* Must match I2C_M_STOP|IGNORE_NAK */

	unsigned short addr;		/* chip address - NOTE: 7bit	*/
					/* addresses are stored in the	*/
					/* _LOWER_ 7 bits		*/
	char name[I2C_NAME_SIZE];
	struct i2c_adapter *adapter;	/* the adapter we sit on	*/
	struct device dev;		/* the device structure		*/
	int init_irq;			/* irq set at initialization	*/
	int irq;			/* irq issued by device		*/
	struct list_head detected;
#if IS_ENABLED(CONFIG_I2C_SLAVE)
	i2c_slave_cb_t slave_cb;	/* callback for slave mode	*/
#endif
	void *devres_group_id;		/* ID of probe devres group	*/
};
```
表示I2C Slave设备

### 5.3.5 数据结构关系
**i2c_adapter与i2c_algorithm**：i2c_adapter对应物理上适配器,而i2c_algorithm对应一套通信方法；适配器需要algorithm提供的通信函数来控制适配器产生特定的访问周期
**i2c_driver与i2c_client**:i2c_driver对应于一套驱动方法,其主要成员函数是probe()、remove()、suspend()、resume()等, 其中struct i2c_device_id形式的id_table是该驱动所支持的I2C设备的ID表。i2c_client对应于真实的物理设备, 每个I2C设备都需要一个i2c client来描述。i2c_driver与i2c client的关系是一对多,一个i2cdriver可以支持多个同类型的i2c client。
**i2c_adapter与i2c_client**:i2c adpater与i2c client的关系与I2C硬件体系中适配器和设备的关系一致,即i2c_client依附于i2c_adpater。一个适配器可以连接多个I2C设备,所以一个i2c_adpater也可以被多个i2c_client依附,
i2c_adpater中包括依附于它的i2c client的链表。
总体关系图如下：
![数据结构关系](image-9.png)

### 5.3.6 编写I2C驱动流程
1）提供I2C适配器的硬件驱动,探测、初始化I2C适配器(如申请I2C的I/O地址和中断号)、驱动CPU控制的I2C适配器从硬件上产生各种信号以及处理I2C中断等。
2) 提供I2C适配器的Algorithm, 用具体适配器的xxx_xfer()函数填充i2c_algorithm的master_xfer指针, 并把i2c_algorithm指针赋值给i2c_adapter的algo指针。
3) 实现I2C设备驱动中的i2c_driver接口,probe()、remove() 、suspend（）和resume()函数指针和i2c device id设备ID表赋值给i2c driver的probe、remove、suspend、resume和id_table指针。
4) 实现I2C设备所对应类型的具体驱动,i2c_driver只是实现设备与总线的挂接, 而挂接在总线上的设备则千差万别。例如,如果是字符设备,就实现文件操作接口,1即实现具体设备read()write()和ioctl()函数等;

## 5.4 I2C API
### 5.4.1 增加和删除adapter
```C
int i2c_add_adapter(struct i2c_adapter *adapter)
void i2c_del_adapter(struct i2c_adapter *adap)
int i2c_add_numbered_adapter(struct i2c_adapter *adap)
```

### 5.4.2 增加和删除i2c_driver
```C
int i2c_register_driver(struct module *owner, struct i2c_driver *driver)
void i2c_del_driver(struct i2c_driver *driver)
```

### 5.4.3 I2C传输发送和接收
```C
int i2c_transfer(struct i2c_adapter *adap, struct i2c_msg *msgs, int num)
static inline int i2c_master_send(const struct i2c_client *client,
                  const char *buf, int count)；
static inline int i2c_master_recv(const struct i2c_client *client,
                  char *buf, int count)；
```

### 5.4.4 Platform相关(以s3c24xx_i2c_driver为例)
```C
static struct platform_driver s3c24xx_i2c_driver = {
	.probe		= s3c24xx_i2c_probe,
	.remove		= s3c24xx_i2c_remove,
	.id_table	= s3c24xx_driver_ids,
	.driver		= {
		.name	= "s3c-i2c",
		.pm	= S3C24XX_DEV_PM_OPS,
		.of_match_table = of_match_ptr(s3c24xx_i2c_match),
	},
};

static int __init i2c_adap_s3c_init(void)
{
	return platform_driver_register(&s3c24xx_i2c_driver);
}
subsys_initcall(i2c_adap_s3c_init);

static void __exit i2c_adap_s3c_exit(void)
{
	platform_driver_unregister(&s3c24xx_i2c_driver);
}
module_exit(i2c_adap_s3c_exit);
```
### 5.4.5 消息相关
```C
struct i2c_msg msg[2];
/*第一条消息是写消息 */
msg[0].addr = client->addr;
msg[0].flags = 0;
msg[0].len = 1;
msg[0].buf = &offs;
/*第二条消息是读消息 */
msg[1].addr = client->addr;
msg[1].flags = I2C_M_RD;
msg[1].len = sizeof(buf);
msg[1].buf = &buf[0];
i2c_transfer(client->adapter, msg, 2);
```
传输行为总结：i2c 总线驱动控制 i2c 适配器产生通信信号，通过 master_xfer()启动一个 i2c 传输，然后通过中断推进 i2c 传输。