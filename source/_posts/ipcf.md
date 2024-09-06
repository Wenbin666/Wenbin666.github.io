---
title: IPCF框架梳理
date: 2024-08-16 17:18:04
tags: ipcf
categories:
 - IPC
---
# 1. IPCF简介
IPCF（Inter-Processor Communication Framework）是恩智浦（NXP）半导体公司设计的一个用于多核处理器之间通信的框架，特别适用于嵌入式系统。
![功能示意](image-1.png)
主要功能：
1. 它支持位于同一芯片或不同芯片上的应用程序，可以通过相关API传输接口进行高效通信。
2. IPCF设计了一组零拷贝API，用户可以直接使用该API来获得最高性能、最低开销和较低的CPU负载。
3. 驱动程序仅在本地存储器域中执行所有写入操作，确保本地和远程共享存储器不会相互干扰。

特性：
* 低延迟：提供高效的核间通信机制，减少通信延迟。
* 小占用空间：优化资源使用，减少内存和CPU负载。
* 灵活性：支持多种操作系统和传输接口（如共享内存）。

# 2. IPCF架构与组成
IPCF的架构由三个层次，多个子模块构成：
![IPCF框架](image-4.png)
**底层硬件**：最底层由中断机制和共享内存，完成硬件依赖。
**中间层**：分为三部分：
硬件驱动（ipc-hw）： 直接与硬件交互，负责驱动硬件操作，如中断和数据操作。
队列机制（ipc-queue）: 负责管理消息，发送方把消息发送到消息队列，接收方去消息队列取数据。
操作系统接口（ipc-os）：负责与操作系统交互，掌控程序的执行流程（中断和任务的调用），向下依赖于硬件驱动，向上服务于功能实现模块。针对不同操作系统提供对应的C语言实现，支持AUTOSAR、FreeRTOS等多个操作系统。
**最上层**：功能实现(ipc-shm),提供对应用程序的软件接口，实现核间通信的逻辑。通过共享内存与中断通知的组合完成消息和数据的传递，支持回调与轮询两种消息访问方式，以及unmanaged和managed两种数据算法。

## 2.1 硬件依赖
IPCF的硬件依赖主要包括：
1. 中断：用于通知对方数据已就绪或读取完成。
2. 共享内存：用于在处理器核之间传递数据。

### 2.1.1 中断依赖
![硬件依赖框架图](image-2.png)
MSCM(Miscellaneous System Control Module)为芯片内杂项控制器，通过写MSCM的IRCP_GR寄存器，可以生成特定的中断信号(Message-Signaled interrupts),这些中断可以发送给特定的core，如A53或者R52。
![alt text](image-3.png)
A53通过写MSCM的IRCP_GR寄存器，可以触发中断到R52, R52也可以通过中断通知A53,通过这种方式就可以以中断的方式发送消息，完成core间消息传递。

### 2.1.2 共享内存
IPCF使用共享内存在处理器核之间传递数据。
![Share Memory 框架](image-5.png)
1. 不同的channel用于不同的驱动模块需求，如SPI和UART等。
2. buffer pool也有不同的内存大小，用于匹配申请的size，减少内存浪费。
3. buffer ptr fifo用于储存申请到的首地址。

# 3. 整体框架与通信流程
整体框架图：
![系统框图](image-7.png)
整体通信流程如下图所示：
![单向通信说明](image-6.png)
1. 发送方先通过shm_acquire_buf申请buffer。
```c
void *ipc_shm_acquire_buf(const uint8_t instance, uint8_t chan_id, uint32_t mem_size);
```
2. 通过shm_tx发送数据，并通过MSCM的IRCP_GR寄存器触发中断通知。
```c
int8_t ipc_shm_tx(const uint8_t instance, int chan_id, void *buf, uint32_t size);
```
3. 接收方收到中断后，进入中断回调，并通过notifiy机制激活相关task。
```C
void ipc_hw_irq_notify(const uint8_t instance)
```
4. 相关task，进行数据处理工作，处理完成通过release_buf释放buffer。
```c
int8_t ipc_shm_release_buf(const uint8_t instance, uint8_t chan_id, const void *buf);
```

# 4. 源码分析
源码位置[ipc-shm](https://github.com/nxp-auto-linux/ipc-shm).
更多细节与函数相关细节可以具体查看。

![source code](image-8.png)

# 5.总结
IPCF特别适用于对实时性和吞吐量要求较高的嵌入式系统，如汽车电子控制单元（ECU）中的多核处理器通信。
通过IPCF，不同的软件组件可以相互通信、协同工作，可以低延迟、高速率完成大数据通信，从而实现整车系统的功能。
