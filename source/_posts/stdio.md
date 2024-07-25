---
title: Uboot serial、stdio 和 console的关系
date: 2022-06-05 21:02:55
tags: stdio
categories:
 - uboot
---
# Uboot serial、stdio 和 console的关系

# 1.前言
uboot启动过程中有两个重要的过程：board_init_f和board_init_r。
serial的初始化以及相关stdio、console操作穿插在这两个函数的执行过程中。
本文目标：
梳理这两阶段中涉及的serial、stdio和console的过程，试图理清其中的原委。
源码地址：https://github.com/ARM-software/u-boot/
# 2. board_init_f阶段serial相关函数
```C
static const init_fnc_t init_sequence_f[] = {
    ...
    init_baud_rate,     /* initialze baudrate settings */
    serial_init,        /* serial communications setup */
    console_init_f,     /* stage 1 init of console */
    ...
}
```
## 2.1 init_baud_rate
```C
// common/board_f.c
init_baud_rate
```
1.	尝试从环境变量baudrate中获取波特率的设定值
2.	若找不到相关的环境变量，则使用默认的波特率配置115200
3.	将最终值赋给全局变量gd->baudrate

## 2.2 serial_init
```C
//drivers/serial/serial.c
int serial_init(void)
{
    gd->flags |= GD_FLG_SERIAL_READY;
    return get_current()->start();
}
```
GD_FLG_SERIAL_READY标志，表明串口已经可用
```C
//找到目前使用的serial dev
static struct serial_device *get_current(void)
{
    struct serial_device *dev;
    if (!(gd->flags & GD_FLG_RELOC))     // RELOC标志还没被置位
        dev = default_serial_console();
    else if (!serial_current)
        dev = default_serial_console();
    else
        dev = serial_current;
    /* We must have a console device */
    if (!dev) {
        panic("Cannot find console\n");
    }
    return dev;
}
```
标志GD_FLG_RELOC在board_r阶段initr_reloc中设置，此时处于uboot启动第一阶段, 还未置位，调用函数default_serial_console();
各个serial驱动都会实现相应函数，由**Makefile**决定具体选择哪一个做default_serial_console
```C
atmel_usart.c|132| __weak struct serial_device *default_serial_console(void)
serial_mtk.c|378| __weak struct serial_device *default_serial_console(void)
serial_mxc.c|258| __weak struct serial_device *default_serial_console(void)
serial_pl102x.c|213| __weak struct serial_device *default_serial_console(void)
serial_pl01x.c|272| __weak struct serial_device *default_serial_console(void)
```
下面以以uboot中serial_pl01x.c为例进行说明（简单清晰）
```C
__weak struct serial_device *default_serial_console(void)
{
    return &pl01x_serial_drv;
}
static struct serial_device pl01x_serial_drv = {
    .name   = "pl01x_serial",
    .start  = pl01x_serial_init,
    .setbrg = pl01x_serial_setbrg,
    .putc   = pl01x_serial_putc,
    .puts   = default_serial_puts,
    .getc   = pl01x_serial_getc,
    .tstc   = pl01x_serial_tstc,
};
```
由此可以推断出：
```C
get_current()->start()  //本质是调用驱动的init函数，此处是pl01x_serial_init
```
init初始化操作(设置波特率和数据位长度，clk和iomux等操作在之前的board_early_init_f函数已经完成)。

# 2.3 console_init_f
```C
int console_init_f(void)
{
    gd->have_console = 1;   //之前已经完成了一个串口的初始化,现在可用了
    console_update_silent(); // 根据环境变量决定是否打开控制台输出并配置gd->flag标志
    // 由宏控制，用于在早期将log打印到buf，目前宏为开启，即不保存早期log
    print_pre_console_buffer(PRE_CONSOLE_FLUSHPOINT1_SERIAL);
    return 0;
}
```
注意：当have_console被设置为1后，基本打印就可以实现了
## 2.3.1 putc
```C
void putc(const char c)
{
    if (!gd)
        return;
    ...
    if (!gd->have_console)
    {
        return pre_console_putc(c);
    }

    if (gd->flags & GD_FLG_DEVINIT) {  //此时GD_FLG_DEVINIT还未置位
        fputc(stdout, c);
    } else {
        pre_console_putc(c);   //打印到buffer
        serial_putc(c);   // 调用串口的输出
    }
}
```
此时调用的还是串口的输出，也就是serial_putc
```C
void serial_putc(const char c)
{
    get_current()->putc(c);    // 又回到了之前分析的！
}
```
## 2.4 总结
*	结构体struct serial_device是在serial层中对串口设备的描述，可以看做串口设备的抽象；
*	struct serial device对下层(SOC层), 对串口属性(如名称)和设备操作函数进行了封装；
*	serial层，提供了诸如serial_puts，serial_getc等外部访问接口，这些接口将调用其私有get_current函数获取一个具体的serial设备实例；并通过调用该serial设备的操作函数puts，getc，最终定位到最底层的SOC serial操作。实现。
其他：
*	在board_init_f阶段，标准stdio设备并未准备好，真正的控制台还未实现；
*	作为替代，uboot提供了一种非标准输入输出的打印(puts)实现。此后的很长一段时间内，都将通过这种方式完成信息输出，直至stdio设备注册到console中。

# 3. board_init_r阶段
```C
static init_fnc_t init_sequence_r[] = {
    ...
    stdio_init_tables,
    serial_initialize,
    stdio_add_devices,
    ...
}
```
## 3.1 stdio_init_tables
```C
int stdio_init_tables(void)
{
    INIT_LIST_HEAD(&devs.list);
    return 0;
}
```
INIT_LIST_HEAD来初始化一个链表
```C
static inline void INIT_LIST_HEAD(struct list_head *list)
{
    list->next = list;
    list->prev = list;
}
```
详细查看下stdio_devs包含哪些内容：
```C
static struct stdio_dev devs;
struct stdio_dev {
    int flags;          /* Device flags: input/output/system    */
    int ext;            /* Supported extensions         */
    char name[32];      /* Device name              */
    int (*start)(struct stdio_dev *dev); /* To start the device */
    int (*stop)(struct stdio_dev *dev);  /* To stop the device */
    void (*putc)(struct stdio_dev *dev, const char c);
    void (*puts)(struct stdio_dev *dev, const char *s);
    int (*tstc)(struct stdio_dev *dev);
    int (*getc)(struct stdio_dev *dev);  /* To get that char */
    void *priv;         /* Private extensions           */
    struct list_head list;
};
```
说明：

* struct stdio_dev代表对标准输入输出stdio设备的描述

* 当有多个stdio设备时，将使用其成员变量list将这些stdio设备链接起来

* stdio_init_tables对该链表进行了初始化

* 由struct list_head list组成的链表可称为stdio设备的维护链表

* u-boot维持着devs这样一个全局变量，它始终指向stdio设备维护链表struct list_head的头

## 3.2 serial_initialize
```C
int serial_initialize(void)
{
    ...
    pl102x_serial_initialize();
    pl01x_serial_initialize();
    ...
    serial_assign(default_serial_console()->name);
    return 0;
}
```
*	serial_initialize调用的函数列表中，末尾serial_assign除外，所有的函数都是weak类型;
*	如果驱动没有具体实现，会被serial.c中的函数serial_null替换, serial_null是空实现。
以pl102x_serial_initialize为例进行说明：
```C
void pl102x_serial_initialize(void)
{
    serial_register(&eserial1_device);
}
void serial_register(struct serial_device *dev)
{
    dev->next = serial_devices;
    serial_devices = dev;
}
```
![](image.png)
```C
static struct serial_device *serial_devices; // 全局变量
struct serial_device {
    char name[16];
    int (*start)(void);
    int (*stop)(void);
    void (*setbrg)(void);
    int (*getc)(void);
    int (*tstc)(void);
    void (*putc)(const char c);
    void (*puts)(const char *s);
    struct serial_device *next;
};
static struct serial_device *serial_current;
```
总结：
*	新加入的serial设备总是插入设备链表的头部。
*	所有注册过的serial设备通过结构成员next链接成一个设备链表
*	全局变量serial_devices指向该链表的表头
*	最后serial_assign根据参数名字name找到具体的serial_device，并将其赋值个serial_current。

## 3.3 stdio_add_devices
```C
// stdio.c
int stdio_add_devices(void)
{
    drv_lcd_init();
    drv_video_init();
    serial_stdio_init();
    drv_usbtty_init();
    ...
    return 0;
}
```
*	函数stdio_add_devices执行stdio设备的注册
*	stdio设备层工作在serail层之上，包括但不限于serial设备，比如标准输出设备可能包括显示器、USB等

## 3.3.1 serial_stdio_init
```C
void serial_stdio_init(void)
{
    struct stdio_dev dev;
    struct serial_device *s = serial_devices;
    while (s) {
        memset(&dev, 0, sizeof(dev));
        strcpy(dev.name, s->name);
        dev.flags = DEV_FLAGS_OUTPUT | DEV_FLAGS_INPUT;  //标准输入输出FLAG
        dev.start = serial_stub_start;
        dev.stop = serial_stub_stop;
        dev.putc = serial_stub_putc;
        dev.puts = serial_stub_puts;
        dev.getc = serial_stub_getc;
        dev.tstc = serial_stub_tstc;
        dev.priv = s;
        stdio_register(&dev);
        s = s->next;
    }
}
```
*	上文已经说明过serial设备链表，它是已经被注册过的soc层serial设备集合
*	serial_devices指向serial设备链表的表头
此处使用while循环来遍历该设备链表并执行如下操作:
1.	将serial设备的设备名拷贝到stdio设备的设备名中
2.	dev.flags配置标准输出输出FLAG
3.	为stdio设备添加ops函数
4.	将serial设备指针赋值给stdio设备结构体的成员变量priv（**很关键，算是serial和stdio设备的纽带**）
5.	注册stdio设备
```C
static void serial_stub_putc(struct stdio_dev *sdev, const char ch)
{
    struct serial_device *dev = sdev->priv;
    dev->putc(ch);
}
```
*	sdev->priv为serial设备指针
*	最终的dev->putc将指向了SOC层中的具体的serial设备输出函数
```C
int stdio_register(struct stdio_dev *dev)
{
    return stdio_register_dev(dev, NULL);
}
int stdio_register_dev(struct stdio_dev *dev, struct stdio_dev **devp)
{
    struct stdio_dev *_dev;
    _dev = stdio_clone(dev);
    if (!_dev)
        return -ENODEV;
    list_add_tail(&_dev->list, &devs.list);
    if (devp)         //传入NULL
        *devp = _dev;
    return 0;
}
```
*	通过函数stdio_clone为dev克隆一个stdio_dev _dev
*	将新建_dev成员变量中的list链表加入到stdio设备总链表的末尾
*	注意这里是单纯的list链， dev结构体自身并末加入链表中
**stdio_add_devices总结：**
1.	为serial设备分配相应的struct_stdio dev实例
2.	将struct_stdio dev的ops函数和serial的ops相关联
3.	利用struct_stdio dev的成员priv将serial设备和代表stdio实例的struct_stdio dev关联起来
4.	将struct_stdio dev的成员list加入全局变量devs的list表，完成最终的seral设备到stdio的注册
5.	后续的stdio设备的访问就可以使用全局变量devs来进行

## 3.4 总结
*	stdio的实现包含3层：stdio层、serail层和soc serial驱动层
*	stdio层、soc serail驱动层都有实体存在，serail层并无内存实体存在，它使用的还是SOC serail分配的内存空间
*	serail层可称为抽象层。它是对相应的具体soc层 serail设备的抽象
![](image-1.png)

# 4. console_init_r函数
```C
static init_fnc_t init_sequence_r[] = {
    ...
    console_init_r,
    ...
}
```
console_init_r根据宏SYS_CONSOLE_IS_IN_ENV的值有两种实现，此处以值为y进行说明,且整体代码较长，以分段方式进行说明：

### 3.4.1 gt->jt相关
```C
int console_init_r(void)
{     /* set default handlers at first */
      gd->jt->getc  = serial_getc;
      gd->jt->tstc  = serial_tstc;
      gd->jt->putc  = serial_putc;
      gd->jt->puts  = serial_puts;
      gd->jt->printf = serial_printf;
      ...
}
```
首先是对gd->jt进行赋值，gd->jt是uboot提供一种方便的方式来动态地调用函数。通过将函数指针存储在跳转表中，可以在运行时根据需要选择要调用的函数。

### 3.4.2 环境变量相关
```C
int console_init_r(void)
{
    char *stdinname, *stdoutname, *stderrname;
    struct stdio_dev *inputdev = NULL, *outputdev = NULL, *errdev = NULL;
    stdinname  = env_get("stdin");
    stdoutname = env_get("stdout");
    stderrname = env_get("stderr");
    if (OVERWRITE_CONSOLE == 0)
    {  //如果OVERWRITE_CONSOLE 为0 ，则进行替换
      inputdev  = console_search_dev(DEV_FLAGS_INPUT,  stdinname);
      outputdev = console_search_dev(DEV_FLAGS_OUTPUT, stdoutname);
      errdev    = console_search_dev(DEV_FLAGS_OUTPUT, stderrname);
    }
    if (inputdev == NULL){
        inputdev  = console_search_dev(DEV_FLAGS_INPUT,  "serial");
    }
    ...
    if (inputdev != NULL) {
          console_doenv(stdin, inputdev);
     }
     ...
}
```
如果在环境变量中通过setenv stdin + console_name就会进行重新配置，如果配置的值找不到，就会通过serial这个名字去搜寻console设备，并将其配置为标准输入输出。

### 3.4.3 console_doenv
```C
static inline void console_doenv(int file, struct stdio_dev *dev)
{
    console_setfile(file, dev);
}
static int console_setfile(int file, struct stdio_dev * dev)
{
      int error = 0;
      if (dev == NULL)
          return -1;
      switch (file) {
      case stdin:
      case stdout:
      case stderr:
          error = console_start(file, dev);
          if (error)
              break;
          stdio_devices[file] = dev;
          switch (file) {
          case stdin:
              gd->jt->getc = getchar;
              gd->jt->tstc = tstc;
              break;
          case stdout:
              gd->jt->putc  = putc;
              gd->jt->puts  = puts;
              gd->jt->printf = printf;
              break;
          }
          break;
      default:        /* Invalid file ID */
          error = -1;
      }
      return error;
}
struct stdio_dev *stdio_devices[] = { NULL, NULL, NULL };
```
•	在上文的stdio_add_devices章节，stdio设备的注册中，只是填充了相关结构体，并未真正启动过注册的设备
•	此处通过console_start，对相应的stdio设备进行初始化(逐步拆解最底层还是调用的serial的init)
•	stdio_devices[]储存真正的stdio设备，即stdin、stdout和stderr，代表当前正在使用的stdio设备

### 3.4.4 GD_FLG_DEVINIT
```C
int console_init_r(void)
{
    ...
    gd->flags |= GD_FLG_DEVINIT;    //表明设备全部init完成
    return 0
}
```
•	当执行gd->fags = GD FLG DEVINIT后，代表此时console控制台已经准备完成并可用
•	控制台提供的getc、tstc、puts等函数在使用过程中都会判断gd->flags的GD_FLG_DEVINIT是否被置位
如之前提到的puts
```C
void puts(const char *s)
{
  ...
  if (gd->flags & GD_FLG_DEVINIT)
  {
      fputs(stdout, s);      //console 建立完全后
  } else {
      serial_puts(s);        //早期串口输出
  }
}
```
具体输出情况，根据不同的标志状态，被分成了多个完全不同的通路！
![](image-2.png)
