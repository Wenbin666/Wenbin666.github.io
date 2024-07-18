---
title: dma
date: 2023-05-27 16:12:42
tags:
---
# DMA简述

# 1. DMA概念
## 1.1 DMA背景
DMA(Direct Memory Access，直接存储器访问)，
背景：相比CPU，memory和外设的速度是非常慢的，因而在memory和memory（或者memory和设备）之间搬运数据，非常浪费CPU的时间，造成CPU无法及时处理一些实时事件。
DMA是一种无需CPU的参与就可以让外设与系统内存之间进行双向数据传输的硬件机制。它可以使系统CPU从实际的I/O数据传输过程中摆脱出来，大大提高系统的吞吐率。
![](image.png)

## 1.2 DMA channel 概念
一个DMA controller可以“同时”进行的DMA传输的个数是有限的，这称作DMA channels。
鉴于总线访问的冲突，以及内存一致性的考量，从物理的角度看，不大可能会同时进行两个（及以上）的DMA传输。
很多时候，DMA channels是DMA controller为了方便，抽象出来的概念，让consumer以为独占了一个channel，实际上所有channel的DMA传输请求都会在DMA controller中进行**仲裁**，进而串行传输；

## 1.3 DMA request lines
DMA传输是由CPU发起的，需要将数据从源地址搬到目的地址。CPU发完指令之后，就全权由DMA控制器负责，除了负责怎么搬之外，还需要知道何时可以开始数据搬运是合理的。
因为，CPU发起DMA传输的时候，并不知道当前是否具备传输条件：比如source设备是否有数据、dest设备的FIFO是否空闲等。只有设备本身知道什么时候是开始搬运的最佳时期！
因此，需要DMA传输的设备和DMA控制器之间，会有几条物理的连接线，也就是request lines。
通常来说，每一个数据收发的节点（称作endpoint），和DMA controller之间，就有一条DMA request line。
比如常用的字符设备UART、I2C和SPI等，都可以配置水位线，当数据超过水位线时，就会给DMA发送信号，来告诉DMA需要进行数据搬运。

## 1.4 传输参数
### 1.4.1 transfer size
表示要传输的byte的个数：在最简单的DMA传输中，仅需配置transfer size.在每一个时钟周期，DMA controller将1byte的数据从一个buffer搬到另一个buffer，直到搬完“transfer size”个bytes后停止。
### 1.4.2 transfer width
DMA传输过程中指定传输的总线宽度：如果是memory to memory的数据传输，用户希望能以总线的最大宽度为单位（32-bit、64-bit等）进行搬运，以提升数据传输的效率；还有一些特定设备。如音频设备，需要每次写入精确的16-bit或者24-bit的数据；
### 1.4.3 burst size
DMA控制器内部可缓存的数据量的大小，称作burst size。（相当于增加一个缓冲区，**用空间换次数**）
memory是源的时候，一次从memory读出一批数据，保存在自己的buffer中，然后再一点点（以时钟为节拍），传输到目的地；
memory是目的地的时候，先将源的数据传输到自己的buffer中，当累计一定量的数据之后，再一次性的写入memory。

## 1.5 Scatter-gather概念
DMA传输一般只能处理在物理上连续的buffer。但在有些场景下，我们需要将一些非连续的buffer拷贝到一个连续buffer中（这样的操作称作scatter gather，聚合）。
对于这种非连续的传输，大多时候都是通过软件，将传输分成多个连续的小块（chunk）。但为了提高传输效率（特别是在图像、视频等场景中），有些DMA controller从硬件上支持了这种操作。
详细说明：
假设在一个系统中有三个模块可以访问memory：CPU、DMA和外设：
**CPU**通过MMU以虚拟地址（VA）的形式访问memory；
**DMA**直接以物理地址（PA）的形式访问memory；
**Device**通过自己的IOMMU以设备地址（DA）的形式访问memory。
![](image-1.png)
假设某个“软件实体”分配并使用了一片存储空间：
![](image-2.png)
该存储空间在CPU视角上（虚拟空间）是连续的，起始地址是va1（实际上它映射到了3块不连续的物理内存上pa1,pa2,pa3）。
如果该软件单纯的以CPU视角访问这块空间（操作va1)，则完全没有问题，因为MMU实现了连续VA到非连续PA的映射。
但是，如果软件经过一系列操作后，要把该存储空间交给DMA控制器，最终由DMA控制器将其中的数据搬移给某个外设的时候，由于DMA控制器只能访问物理地址，只能以“不连续的物理内存块”为单位递交，此时就需要scatterlist这种数据结构的参与！
**本质**：scatterlist是各种不同地址映射空间（PA、VA、DA、）的媒介（物理地址是真实存在的，因而可以作为通用语言)，借助它这些映射空间才能相互转换。

# 2. DMA engine使用
## 2.1 传输基本概念
从方向上来说，DMA传输可以分为4类：memory到memory、memory到device、device到memory以及device到device。
从Linux kernel的视角看，外设都是slave，因此称这些有device参与的传输（MEM2DEV、DEV2MEM、DEV2DEV）为Slave-DMA传输。
而另一种memory到memory的传输，被称为Async TX：主要原因是因为Linux为了方便基于DMA的memcpy、memset等操作，在dma engine之上，封装了一层更为简洁的API，这种API就是Async TX API（以async_开头，例如async_memcpy、async_memset、async_xor等），因为这种操作是访问不同的memory相对于访问同一块memory来说所以称之为Async。

## 2.2 使用DMA传输步骤
对设备驱动的编写者来说，要基于dma engine提供的Slave-DMA（有设备参与的）API进行DMA传输的话，需要如下的操作步骤：
1）申请一个DMA channel。
2）根据设备（slave）的特性，配置DMA channel的参数。
3）要进行DMA传输的时候，获取一个用于识别本次传输（transaction）的描述符（descriptor）。
4）将本次传输（transaction）提交给dma engine并启动传输。
5）等待传输（transaction）结束。
后续传输不需要重新申请通道，直接获取传输描述符继续重复后续步骤即可。

### 2.2.1 申请DMA channel
任何device在开始DMA传输之前，都要申请一个DMA channel：
```C
struct dma_chan *dma_request_chan(struct device *dev, const char *name);
```
该接口会返回绑定在指定设备（dev）上名称为name的dma channel。
```C
void dma_release_channel(struct dma_chan *chan);
```
申请得到的dma channel可以在不需要使用的时候通过release释放掉。

### 2.2.2 配置DMA channel 参数
```C
int dmaengine_slave_config(struct dma_chan *chan, struct dma_slave_config *config)；
```
device申请到一个为自己使用的DMA channel后，需要根据自身的实际情况，以及DMA controller的能力，对该channel进行一些配置。

### 2.2.3 获取传输描述符
DMA传输属于异步传输，在启动传输之前，slave driver需要将此次传输的一些信息（src/dst的buffer、传输的方向等）提交给dma engine，dma engine确认无误后，返回一个描述符（由struct dma_async_tx_descriptor抽象）。
此后，device就可以以该描述符为单位，控制并跟踪此次传输。
根据具体的传输模式不同，可以使用如下三个API获取传输描述符：
```C
struct dma_async_tx_descriptor *dmaengine_prep_slave_sg( struct dma_chan *chan, struct scatterlist *sgl, unsigned int sg_len, enum dma_data_direction direction, unsigned long flags);
struct dma_async_tx_descriptor *dmaengine_prep_dma_cyclic( struct dma_chan *chan, dma_addr_t buf_addr, size_t buf_len, size_t period_len, enum dma_data_direction direction);
struct dma_async_tx_descriptor *dmaengine_prep_interleaved_dma( struct dma_chan *chan, struct dma_interleaved_template *xt, unsigned long flags);
```
dmaengine_prep_slave_sg用于在“scatter gather buffers”列表和总线设备之间进行DMA传输
dmaengine_prep_dma_cyclic常用于音频等场景中，在进行一定长度的dma传输（buf_addr&buf_len）的过程中，每传输一定的byte（period_len），就会调用一次传输完成的回调函数
dmaengine_prep_interleaved_dma可进行不连续的、交叉的DMA传输，通常用在图像处理、显示等场景中

### 2.2.4 启动传输
```C
dma_cookie_t dmaengine_submit(struct dma_async_tx_descriptor *desc)；
void dma_async_issue_pending(struct dma_chan *chan);
```
获取传输描述符之后，device可以通过dmaengine_submit接口将该描述符放到传输队列上，然后调用dma_async_issue_pending接口，启动传输。

### 2.2.5 等待传输结束
```C
static inline enum dma_status dma_async_is_tx_complete(struct dma_chan *chan, dma_cookie_t cookie, dma_cookie_t *last, dma_cookie_t *used)；
```
传输请求被提交之后，device可以通过回调函数获取传输完成的消息，也可以通过dma_async_is_tx_complete等API，测试传输是否完成。

### 2.2.6 暂停终止传输
```C
static inline int dmaengine_pause(struct dma_chan *chan)；
static inline int dmaengine_resume(struct dma_chan *chan)；
static inline int dmaengine_terminate_all(struct dma_chan *chan)；
static inline int dmaengine_terminate_async(struct dma_chan *chan)；
static inline int dmaengine_terminate_sync(struct dma_chan *chan)；
```
# 3. 相关数据结构说明
# 3.1 dma_slave_config
```C
/* include/linux/dmaengine.h */
struct dma_slave_config {
        enum dma_transfer_direction direction;
        phys_addr_t src_addr;
        phys_addr_t dst_addr;
        enum dma_slave_buswidth src_addr_width;
        enum dma_slave_buswidth dst_addr_width;
        u32 src_maxburst;
        u32 dst_maxburst;
        bool device_fc;
        unsigned int slave_id;
};
```
* direction: 传输的方向（DMA_MEM_TO_MEM、DMA_MEM_TO_DEV、DMA_DEV_TO_MEM和DMA_DEV_TO_DEV）
* src_addr: 源地址
* dst_addr: 目的地址
* src_addr_width:源地址宽度
* dst_addr_width:目的地址宽度
* src_maxburst: 最大可传输的burst size，单位是src_addr_width
* dst_maxburst: 最大可传输的burst size，单位是dst_addr_width
* device_fc: 当外设是Flow Controller的时候，需要将该字段设置为true
* slave_id : 用于告知dma controller自己是谁（和request line对应）
* slave_addr_type: DMA slave 地址类型

## 3.2 dma_async_tx_descriptor
```C
struct dma_async_tx_descriptor {
         dma_cookie_t cookie;
         enum dma_ctrl_flags flags; /* not a 'long' to pack with cookie */
         dma_addr_t phys;
         struct dma_chan *chan;
         dma_cookie_t (*tx_submit)(struct dma_async_tx_descriptor *tx);
         int (*desc_free)(struct dma_async_tx_descriptor *tx);
         dma_async_tx_callback callback;
         void *callback_param;
         struct dmaengine_unmap_data *unmap;
        ...
};
```
* Cookie : 整型数字，用于追踪本次传输；
* flag:
    + DMA_CTRL_REUSE: 表示描述符可重用，直到被清除或释放；
    + DMA_CTRL_ACK: 如果值为0，代表暂时不能被重复使用；
* phys: 描述符的物理地址
* chan ：对应DMA的channel
* tx_submit: device driver提供的回调函数，用于把描述符提交到待传输列表
* desc_free: 释放该描述符的回调函数
* callback：此操作完成的回调函数
* callback_result: 回调函数传递的结果
* callback_param: 回调函数要传递的参数

# 4. 虚拟地址、物理地址和总线地址
在DMA API中涉及好几个地址的概念：
内核通常使用的地址是虚拟地址，我们调用kmaloc0、maloc或者类似的接口返回的地址都是虚拟地址；虚拟内存系统(TLB、页表等)将虚拟地址(程序角度)翻译成物理地址(CPU角度)。
硬件设备上的寄存器等资源，是按照物理地址来管理的。通过/proc/iomem，可以看到这些和设备IO相关的物理地址，驱动并不能直接使用这些物理地址，必须首先通过ioremap()接口将这些物理地址映射到内核虚拟地址空间上去。
IO设备使用第三种地址:总线地址。如果设备可以通过DMA执行读与系统内存的操作，这些情况下，设备使用的地址就是总线地址，在某些系统中，总线地址与CPU物理地址相同，但一般来说它们不是。iommu和host_bridge可以在物理地址和总线地址之间进行映射。
![](image-3.png)

# 5. DMA mapping分类
![](image-4.png)
可以分为一致性DMA映射和流式DMA映射
