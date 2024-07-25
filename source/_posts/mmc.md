---
title: mmc 基本概念与驱动简述
date: 2024-04-18 21:46:58
tags: mmc
categories:
 - Peripheral
---

# mmc 基本概念与驱动简述

# 1. 基本概念
MMC是Multi-MediaCard的简称，本质上它是一种用于固态非易失性存储的内存卡（memory card）规范，定义了诸如卡的形态、尺寸、容量、电气信号、和主机之间的通信协议等方方面面的内容。
* MMC强调的是多媒体存储（MM，MultiMedia）；
* SD强调的是安全和数据保护（S，Secure）；
* SDIO是从SD演化出来的，强调的是接口（IO，Input/Output），不再关注另一端的具体形态。
![mmc 硬件框图](image.png)

# 2. mmc卡的规范
![](image-1.png)
卡的规范主要规定卡的形状、物理尺寸、管脚，内部block组成、寄存器等等，以eMMC为例：
* Memory core，存储介质，一般是NAND flash、NOR flash；
* Memory core interface，管理存储介质的接口，用于访问（读、写、擦除等操作）存储介质；
* Card interface（CMD、CLK、DATA），总线接口，外界访问卡内部存储介质的接口，和具体的管脚相连；
* Card interface controller，将总线接口协议转换为Memory core interface的形式，用于访问内部存储介质；
* Power模块，提供reset、上电检测等功能；
* 寄存器（Card interface controller的左侧小矩形），用于提供卡的信息、参数、访问控制等功能。
卡管脚：
* CLK有一条，提供同步时钟，可以在CLK的上升沿（或者下降沿，或者上升沿和下降沿）采集数据；
* CMD有一条，用于传输双向的命令。
* DATA用于传输双向的数据，根据MMC的类型，可以有一条（1-bit）、四条（4-bit）或者八条（8-bit）

# 3. Sectors、Blocks和Segments概念
**Sectors**(扇区)是存储设备访问的基本单位。对磁盘、NAND等块设备来说，Sector的size是固定的，例如512、2048等。
对存储类的MMC设备来说，理应有固定size的sector。但因为有MMC协议的封装，且存在host驱动以及块设备驱动，所以不需要关注物理的size，需要关注的就是bus上的数据传输单位。
**Blocks**(块)是数据传输的基本单位，是VFS抽象出来纯软件的概念。它必须是2的整数倍、不能大于Sectors的单位、不能大于page的长度，一般为512B、2048B或者4096B。
MMC设备可以支持一定范围内的任意的Block size，主要由两个因素决定：
a）host controller的能力，这反映在struct mmc_host结构的max_blk_size字段上。
b）卡的能力，这可以通过MMC command从卡的CSD（Card-Specific Data）寄存器中读出。
**Segments**(段):每个内存区域描述（物理连续的一段内存，可以是一个page，也可以是page的一部分），就称作Segment。一个Segment包含多个相邻扇区。

# 4. mmc总线规范
1）物理信号有CLK、CMD和DATA三类。
2）电压范围为1.65V和3.6V，根据工作电压的不同，MMC卡可以分为两类：
￮	High Voltage MultiMedia Card，工作电压为2.7V\~3.6V。
￮	Dual Voltage MultiMedia Card，工作电压有两种，1.70V\~1.95V和2.7V\~3.6V，CPU可以根据需要切换。
3）数据传输的位宽（data bus width mode）是允许动态配置的，包括1-bit (默认)模式、4-bit模式和8-bit模式。
￮	不使用的数据线，需要保持上拉状态。
￮   由于数据线宽度是动态可配的，这要求CPU可以动态的enable/disable数据线的那些上拉电阻。
4）MMC规范定义了CLK的频率范围，包括0-20MHz、0-26MHz、0-52MHz等几种，结合数据线宽度，基本决定了MMC的访问速度。
5）总线规范定义了一种简单的、主从式的总线协议，MMC卡为从机（slave），CPU为主机（Host）。
6）协议规定了三种可以在总线上传输的信标（token）：
￮	Command，Host通过CMD线发送给Slave的，用于启动（或结束）一个操作；
￮	Response，Slave通过CMD线发送给Host，用于回应Host发送的Command；
￮	Data，Host和Slave之间通过数据线传输的数据。方向可以是Host到Slave，也可以是Slave到Host。数据线的个数可以是1、4或者8。在每一个时钟周期，每根数据线上可以传输1bit或者2bits的数据。
7）一次数据传输过程，需要涉及所有的3个信标。一次数据传输的过程也称作Bus Operation，根据场景的不同，MMC协议规定了很多类型的Bus Operation。

# 5. 软件架构framework
![mmc framwork](image-2.png)

MMC core位于中间，是MMC framework的核心实现，负责抽象host、bus、card等软件实体，负责向底层提供统一、便利的编写Host controller driver的API；

MMC host controller driver位于底层，基于MMC core提供的框架，驱动具体的硬件（MMC controller）；

MMC card driver位于最上面，负责驱动MMC core抽象出来的虚拟的card设备，并对接内核其它的framework（例如块设备、TTY、wireless等），实现具体的功能。

host，Host controller，提供访问card的寄存器、检测card的插拔、读写card等操作方法。从设备模型的角度看，host会检测卡的插入，并向bus注册MMC card设备；

bus，是MMC bus的虚拟抽象，以标准设备模型的方式，收纳MMC card（device）以及对应的MMC driver；

card，抽象具体的MMC卡，由对应的MMC driver驱动（如存储卡、WIFI卡等等）。

# 6. 工作流程
![](image-3.png)
主要分为mmc core、host driver和card driver。

# 7. mmc host驱动
MMC的host driver，是用于驱动MMC host控制器的程序，位于drivers/mmc/host目录。
可以通过以下步骤编写驱动：
1）调用mmc_alloc_host，分配一个struct mmc_host类型的变量，用于描述某一个具体的MMC host控制器。
2）根据host控制器的硬件特性，填充struct mmc_host的各个字段，例如MMC类型、电压范围、操作函数集等。
3）调用mmc_add_host接口，将正确填充的MMC host注册到MMC core中。

## 7.1 主要数据结构
### 7.1.1 mmc_host
```C
// /include/linux/mmc/host.h
struct mmc_host {
    struct device       *parent;     //指向该MMC host的父设备，一般是注册该host的那个platform设备；
    struct device       class_dev;   //该MMC host在设备模型中作为一个“设备”的体现
    const struct mmc_host_ops *ops;  //保存了该MMC host有关的操作函数集
    struct mmc_pwrseq   *pwrseq; //保存了该MMC host电源管理有关的操作函数集
    unsigned int        f_min;   //host支持的最小频率
    unsigned int        f_max;   //host支持的最大频率
    unsigned int        f_init;  //host的初始频率
    u32         ocr_avail;       //该MMC host可支持的操作电压范围 Operating Conditions Register
    u32         ocr_avail_sdio; /* SDIO-specific OCR */
    u32         ocr_avail_sd;   /* SD-specific OCR */
    u32         ocr_avail_mmc;  /* MMC-specific OCR */
    struct wakeup_source    *ws;        /* Enable consume of uevents */
    u32         max_current_330;
    u32         max_current_300;
    u32         max_current_180;
...
};
```
### 7.1.2 mmc_host_ops
```C
struct mmc_host_ops {
    void    (*post_req)(struct mmc_host *host, struct mmc_request *req, int err);
    void    (*pre_req)(struct mmc_host *host, struct mmc_request *req);
    void    (*request)(struct mmc_host *host, struct mmc_request *req);
    int (*request_atomic)(struct mmc_host *host, struct mmc_request *req);
    void    (*set_ios)(struct mmc_host *host, struct mmc_ios *ios);
    int (*get_ro)(struct mmc_host *host);
...
};
```
包含数据传输有关的函数、总线参数的配置以及卡状态的获取函数等

### 7.1.3 mmc_pwrseq
mmc_pwrseq 结构体用于描述和控制MMC主机的电源序列(Power Sequence),以及一些简单的pwrseq策略。
```C
struct mmc_pwrseq {
    const struct mmc_pwrseq_ops *ops;
    struct device *dev;
    struct list_head pwrseq_node;
    struct module *owner;
};
```
### 7.1.4 mmc_ios
```C
struct mmc_ios {
      unsigned int    clock;          /* clock rate */
      unsigned short  vdd; //卡的供电电压
      unsigned int    power_delay_ms;     /* waiting for stable power */
      unsigned char   bus_mode;       /* command output mode */
      unsigned char   chip_select;        /* SPI chip select */
      unsigned char   power_mode;     /* power supply mode */
      unsigned char   bus_width;      /* data bus width */
...
};
```
保存了MMC总线当前的配置情况，包括时钟频率、卡电压，信号模式等。

### 7.1.5 mmc_request
```C
struct mmc_request {
    struct mmc_command  *sbc;       /* SET_BLOCK_COUNT for multiblock */
    struct mmc_command  *cmd;
    struct mmc_data     *data;
    struct mmc_command  *stop;
    struct completion   completion; //用于等待此次传输完成
    struct completion   cmd_completion;
    void            (*done)(struct mmc_request *);/* completion function */
    void            (*recovery_notifier)(struct mmc_request *);
    struct mmc_host     *host;
    bool            cap_cmd_during_tfr;
    int         tag;
};
```
struct mmc_request封装了一次传输请求。

### 7.1.6 mmc_command
```C
struct mmc_command {
    u32         opcode;   //Command的操作码，用于标识该命令是哪一个命令
    u32         arg;      //Command可能会携带参数
    u32         resp[4]; //Command发出后，如果需要应答，结果保存在resp数组中
    unsigned int        flags;      /* expected response type */
    unsigned int        retries;    /* max number of retries */
    int         error;      /* command error */
...
};
```
struct mmc_command结构抽象了一个MMC command, 包含操作码、参数等。

### 7.1.7 mmc_data
```C
struct mmc_data {
    unsigned int        timeout_ns; /* data timeout (in ns, max 80ms) */
    unsigned int        timeout_clks;   /* data timeout (in clocks) */
    unsigned int        blksz;      /* data block size */
    unsigned int        blocks;     /* number of blocks */
    unsigned int        blk_addr;   /* block address */
    int         error;      /* data error */
    unsigned int        flags;
    unsigned int        bytes_xfered;
    struct mmc_command  *stop;      /* stop command */
    struct mmc_request  *mrq;       /* associated request */
    unsigned int        sg_len;     /* size of scatter list */
    int         sg_count;   /* mapped sg entries */
    struct scatterlist  *sg;        /* I/O scatter list */
    s32         host_cookie;    /* host private data */
};
```
struct mmc_data结构包含了数据传输有关的内容。
## 7.2 主要API

### 7.2.1 分配host
```C
struct mmc_host *mmc_alloc_host(int extra, struct device *);
void mmc_free_host(struct mmc_host *);
```
mmc_alloc_host，动态分配一个struct mmc_host变量。extra是私有数据的大小，可通过host->private指针访问（也可通过mmc_priv接口直接获取）。
mmc_free_host执行相反动作。

### 7.2.2 新增host
```C
int mmc_add_host(struct mmc_host *);
void mmc_remove_host(struct mmc_host *);
```
mmc_add_host，将已初始化好的host变量注册到kernel中。mmc_remove_host执行相反动作。

### 7.2.3 电源管理相关
```C
int mmc_power_save_host(struct mmc_host *host);
int mmc_power_restore_host(struct mmc_host *host);
```
从mmc host的角度进行电源管理，进入/退出power save状态。

### 7.2.4 detect_change
```C
void mmc_detect_change(struct mmc_host *, unsigned long delay);
```
当host driver检测到总线上的设备有变动的话（例如卡的插入和拔出等），需要调用这个接口，让MMC core帮忙做后续的工作。

### 7.2.5 request_done
```C
void mmc_request_done(struct mmc_host *, struct mmc_request *);
```
当host driver处理完成一个mmc request之后,需要调用该函数通知MMC core，MMC core会进行一些善后的操作，例如校验结果、调用mmc request的.done回调等等。

### 7.2.6 IRQs
```C
static inline void mmc_signal_sdio_irq(struct mmc_host *host)
void sdio_run_irqs(struct mmc_host *host);
```
对于SDIO类型的总线，这两个函数用于操作SDIO irqs。

### 7.2.7 电压相关
```C
int mmc_regulator_get_ocrmask(struct regulator *supply);
int mmc_regulator_set_ocr(struct mmc_host *mmc,struct regulator *supply, unsigned short vdd_bit);
int mmc_regulator_set_vqmmc(struct mmc_host *mmc, struct mmc_ios *ios);
int mmc_regulator_get_supply(struct mmc_host *mmc);
```
mmc_regulator_get_ocrmask可根据传入的regulator指针，获取该regulator支持的所有电压值。
mmc_regulator_set_ocr用于设置host controller为某一个操作电压
mmc_regulator_set_vqmmc可根据struct mmc_ios信息，调用regulator framework的接口，设置vqmmc的电压
mmc_regulator_get_supply可以帮忙从dts的vmmc、vqmmc属性值中，解析出对应的regulator指针
# 8. MMC驱动编写
## 8.1 mmc_hos的填充和注册
编写MMC host驱动的所有工作，都是围绕struct mmc_host结构展开的。
在对应的platform driver的probe函数中，通过mmc_alloc_host分配一个mmc host后，根据controller的实际情况，填充对应的字段。
mmc host中大部分和controller能力/特性有关的字段，可以通过dts配置，举例如下：
```C
/* arch/arm/boot/dts/exynos5420-peach-pit.dts */
&mmc_1 {
        status = "okay";
        num-slots = <1>;
        non-removable;  // mmc 标准字段
        cap-sdio-irq;   // mmc 标准字段
        keep-power-in-suspend;
        clock-frequency = <400000000>;  // mmc 标准字段
        samsung,dw-mshc-ciu-div = <1>;
        samsung,dw-mshc-sdr-timing = <0 1>;
        samsung,dw-mshc-ddr-timing = <0 2>;
        pinctrl-names = "default";
        pinctrl-0 = <&sd1_clk>, <&sd1_cmd>, <&sd1_int>, <&sd1_bus1>,
                    <&sd1_bus4>, <&sd1_bus8>, <&wifi_en>;
        bus-width = <4>; // mmc 标准字段
        cap-sd-highspeed; // mmc 标准字段
        mmc-pwrseq = <&mmc1_pwrseq>; // mmc 标准字段
        vqmmc-supply = <&buck10_reg>; // mmc 标准字段
};
```
## 8.2 数据传输的实现
填充struct mmc_host变量的过程中，工作量最大的，就是对struct mmc_host_ops的实现，因为所有MMC host的操作逻辑都封在这里，可以参考具体的mmc驱动实例进行查看。
