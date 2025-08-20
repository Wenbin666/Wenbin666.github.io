---
title: 等待队列和工作队列
date: 2025-08-20 21:10:52
tags: kernel
categories:  
 - Scheduler
---

# 等待队列和工作队列

## 1. 等待队列（Wait Queue）

**作用**
用于实现线程的阻塞和唤醒，通常用于同步场景，比如：
- 一个线程需要等待某个条件满足（如数据到达、资源可用）。
- 当条件不满足时，线程进入睡眠状态，释放CPU资源；当条件满足时，内核唤醒线程继续执行。

**特点**
- 基于事件阻塞：线程主动睡眠，直到被外部事件唤醒（如中断、另一个线程的信号）。
- 不占用CPU：睡眠的线程不参与调度，直到被唤醒。
- 常用于同步：如设备驱动中等待硬件中断、信号量实现等。

**典型使用场景**

```c
// 定义等待队列
DECLARE_WAIT_QUEUE_HEAD(wq);

// 等待条件
wait_event_interruptible(wq, condition);

// 唤醒队列（通常在中断或另一个线程中调用）
wake_up(&wq);
```

## 2. 工作队列（Work Queue）

**作用**
用于延迟执行任务，将工作推送到内核线程中异步处理，通常用于：
- 中断下半部（Bottom Half）处理，避免在中断上下文中执行耗时操作。
- 需要调度上下文的任务（如需要睡眠的操作）。

**特点**
- 异步执行：任务被提交到工作队列后，由内核线程在后台调度执行。
- 可睡眠：工作队列运行在进程上下文，可以调用可能睡眠的函数（如 kmalloc、mutex_lock）。
- 动态调度：内核提供默认工作队列（如 system_wq），也支持创建自定义工作队列。

**典型使用场景**

```c
// 定义工作项
struct work_struct my_work;

// 初始化工作项
INIT_WORK(&my_work, my_work_handler);

// 提交到工作队列（可能在中断上下文中调用）
schedule_work(&my_work);
```

## 3. 区别与联系

**区别**

| **特性**      | **等待队列（Wait Queue）** | **工作队列（Work Queue）** |
|-------------|----------------------|----------------------|
| **目的**      | 线程阻塞等待条件满足           | 异步执行延迟任务             |
| **执行上下文**   | 直接关联调用线程             | 由内核线程异步执行            |
| **是否占用CPU** | 睡眠时不占用               | 工作时占用（由内核线程调度）       |
| **是否可睡眠**   | 本身就是睡眠状态             | 工作处理函数可以睡眠           |
| **典型应用**    | 同步（如驱动阻塞读写）          | 异步（如中断下半部、延迟任务）      |
| **触发方式**    | 主动睡眠 + 被动唤醒（wake_up） | 主动提交（schedule_work）  |



**联系**
- 两者均属于内核的任务调度机制，可能结合使用。例如：
  - 在中断处理中，通过工作队列推迟耗时任务，而等待队列用于唤醒阻塞的线程。
  - 工作队列的处理函数内部可能使用等待队列（如等待某个资源）。


---

## 4. 如何选择？

- 如果需要 等待某个条件（如数据就绪），用 等待队列。
- 如果需要 异步执行任务（如中断后处理），用 工作队列。

## 5. 延迟工作项（Delayed Work，工作队列的一种）
- 普通工作队列（INIT_WORK）会尽快调度任务，而 INIT_DELAYED_WORK 允许指定一个 延迟时间，任务会在未来的某个时间点被执行。
- 典型用途：
  - 需要延迟执行的异步任务（如硬件去抖、定时操作）。
  - 替代简单的内核定时器（timer），但需要在进程上下文中运行（可睡眠）。

  关键区别：INIT_WORK vs INIT_DELAYED_WORK

| 特性 | 标准工作 (Work) | 延迟工作 (Delayed Work) |
| :--- | :--- | :--- |
| **初始化宏** | `INIT_WORK(_work, _func)` | `INIT_DELAYED_WORK(_dwork, _func)` |
| **数据结构** | `struct work_struct` | `struct delayed_work` (内含一个 `work_struct`) |
| **调度函数** | `schedule_work(struct work_struct *work)` | `schedule_delayed_work(struct delayed_work *dwork, unsigned long delay)` |
| **执行时机** | 尽快将工作加入队列，由内核线程在**下一个机会**执行。 | 在指定的 **`delay` 个 jiffies** 之后，再将工作加入队列执行。 |
| **典型应用** | 处理中断下半部、不需要延时的异步任务。 | 需要等待一段时间后再执行的任务，例如轮询、超时处理、避免频繁操作。 |
| **示例延迟** | 不适用 | `HZ` (1秒)， `HZ/10` (100毫秒)， `2*HZ` (2秒) |
| **是否可取消** | 是，使用 `cancel_work_sync()` | 是，使用 `cancel_delayed_work_sync()` |

**使用示例**
```c
#include <linux/workqueue.h>

// 定义延迟工作项和回调函数
struct delayed_work my_delayed_work;
void my_delayed_handler(struct work_struct *work) {
    printk("Delayed work executed!\n");
}

// 初始化延迟工作项（通常在模块初始化时）
INIT_DELAYED_WORK(&my_delayed_work, my_delayed_handler);

// 提交延迟任务（延迟 1 秒执行）
schedule_delayed_work(&my_delayed_work, HZ); // HZ = 1 秒的时钟节拍

// 取消延迟任务（如果需要）
cancel_delayed_work(&my_delayed_work);
```

**底层实现**
```c
- delayed_work 本质是 work_struct 的扩展，内嵌了一个定时器（timer）：
struct delayed_work {
    struct work_struct work;
    struct timer_list timer;
};
```
- 当定时器到期后，内核会将工作项提交到工作队列中执行。
- 工作队列会在适当的时候（如空闲时）从队列中取出工作项并执行。

**适用场景**
- 硬件去抖：如按键中断后延迟读取状态，避免抖动。
- 定时任务：替代 timer 实现更复杂的逻辑（因工作队列可睡眠）。
- 资源延迟释放：如网络驱动中延迟清理缓冲区。
