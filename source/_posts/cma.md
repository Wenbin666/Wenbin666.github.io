---
title: CMA概述
date: 2024-08-27 16:12:42
tags: cma
categories:  
  - memory management
---

# 1. CMA简介和特性
## 1.1 简介
CMA（Contiguous Memory Allocator，连续内存分配）是内存管理子系统中的一个模块，**负责物理地址连续的内存分配**。
启动过程中，系统一般会从整个memory中配置一段连续内存用于CMA，然后内核其他的模块可以通过CMA的接口API进行连续内存的分配。CMA的核心并不是设计精巧的算法来管理地址连续的内存块，实际上它的底层还是依赖内核伙伴系统这样的内存管理机制。
**理解**：CMA处于需要连续内存块的其他内核模块（例如DMA mapping framework）和内存管理模块之间的一个中间层模块。
其他内核模块(需要连续内存块) <---> CMA <---> 内存管理模块

## 1.2 功能特性
1、解析DTS或者命令行中的参数，确定CMA内存的区域(称为CMA area)
2、提供cma_alloc和cma_release两个接口函数用于分配和释放CMA pages
3、记录和跟踪CMA area中各个pages的状态
4、调用伙伴系统接口，进行真正的内存分配

# 2.为什么需要CMA？
## 2.1 连续物理地址的需求
1. huge page模块对物理地址连续的需求
以ARM64为例，其内存管理单元可以支持4k、64K、2M或者更大的page size等多个页面大小，但Linux内核总是倾向使用最小的page size --4K。Page size大于4K的page统称为“huge page”巨页。
对于一个2M的huge page，MMU会把一个连续的2M的虚拟地址mapping到连续的2M的物理地址上，且这2M size的物理地址段必须是由512个地址连续的4k page frame组成。
2. 各种外设对联系内存的分配需求
如手机视频功能、camera功能，这类驱动都需要大块的内存，而且有DMA用来进行外设和大块内存之间的数据交换。
在一些嵌入式设备中（无IOMMU，且DMA不具备scatter-getter功能），此时驱动分配的大块内存(DMA buffer)必须是物理地址连续的。
补充：
huge page的连续内存需求和驱动DMA buffer还是有不同的，如在对齐要求上，一个2M的huge page，其底层的2M的物理页面的首地址需要对齐在2M上。而DMA buffer不会有这么高的对齐要求。
本文说明的CMA主要是为**设备驱动准备的**，huge page相关的内容并不包含在内。

实例：
像素是1300W的手机相机，一个像素需要3B，拍摄一幅图片需要的内存为：1300W x 3B ＝ 26MB。

通过内存管理系统分配26M的内存，其实是一件压力比较的大的事情。若在系统启动之初，伙伴系统中的大块内存比较大，分配26M相对容易，但随着系统的运行，内存不断的分配和释放，内存碎片越来越多，这时分配地址连续的大块内存变得不是那么的容易。
此时会面临两个选择：

1. 在启动时分配用于视频采集的DMA buffer
这种方式是可靠的，在camera的使用的时候肯定会有内存可用。但有一个缺点，即当照相机不使用时，预留的那些DMA BUFFER的内存实际上是浪费了。

2. 当实际使用camera设备的时候分配DMA buffer
这种选择不会浪费内存，但是不可靠，随着内存碎片化，大的、连续的内存分配变得越来越困难，一旦内存分配失败，camera功能就会失效。

思考：如果有这样一种内存就好了！
1. 可以提前预留，如果camera不使用时，可以给其他模块使用。
2. 当camera需要使用时，其他模块可以马上释放这部分连续内存，并给camera使用。

CMA内存就具有这样的功能！
1. 当前驱动没有分配使用的时候，这些memory可以内核的被其他的模块使用（当然有一定的要求）。
2. 当驱动分配CMA内存后，那些被其他模块使用的内存需要吐出来，形成物理地址连续的大块内存，给具体的驱动来使用。

# 3. CMA框架简介
## 3.1 MemBlock和Buddy system
在介绍CMA框架前，先简单介绍下MemBlock和Buddy System(伙伴系统)。
memblock：Linux内核在启动后用于管理DRAM空间的一种抽象结构，它是bootmem分配器的升级版本。
在系统启动初期，当buddy系统（伙伴系统）和slab分配器等高级内存管理器尚未初始化时，它负责执行内存管理和分配的任务。一旦这些高级内存管理器初始化完成，memblock分配器会被释放，其内存管理职责将移交给它们。（有点earlycon的赶脚）

伙伴系统：是一种将**物理内存划分为不同大小的连续块**，并以这些块为基本单位进行内存**分配和释放**的内存管理**算法**。
Buddy System将内存按照页面大小进行划分，每个页面通常是4KB。内存块的大小是2的幂次方个页面，例如1个页面（4KB）、2个页面（8KB）、4个页面（16KB）等，直到系统定义的最大块大小（如4MB）。
**核心思想**：系统使用多个链表（通常为11个）来管理这些不同大小的内存块，每个链表对应一种大小的内存块。
**分配**：当系统需要分配内存时，Buddy System会尽量从**较小的内存块链表中查找空闲块**。如果没有找到合适大小的空闲块，它会从**更大的内存块链表**中取出一个块并**分割成两个伙伴块**，其中一个用于分配，另一个保持空闲。这个过程会一直重复，直到找到合适大小的内存块。
**释放**：当内存块被释放时，Buddy System会检查其伙伴块是否也处于空闲状态。如果两个伙伴块都空闲，则将它们合并成一个更大的内存块，并尝试将其放入更大的内存块链表中。这个过程会一直重复，直到无法再合并为止。
在/proc/buddyinfo/中可以看到目前buddy的状态:
![buddyinfo](image-1.png)
Node 0：表示系统中的第一个内存节点（Node），在多处理器系统内存可能会被分割成多个节点以提高访问效率，在许多小型系统（如Raspberry Pi）上，通常只有一个内存节点。
zone Normal：这表示内存区域是“Normal”区域，物理内存划分为几个区域（Zone），每个区域用于不同的目的。通常，“Normal”区域包含的是可以直接映射到内核地址空间的内存。
接下来的数字（81 17 15 1 2 2 2 2 2 4 30）代表了不同大小内存块的空闲数量：
81：表示大小为1个页面(4K)的内存块有81个空闲。
17：表示大小为2个页面(8K)的内存块有17个空闲。
15：表示大小为4个页面(16K)的内存块有15个空闲。

## 3.2 CMA内存管理参数配置
![CMA在内存管理中的位置](image.png)
CMA主要用于解决驱动的内存分配问题，但驱动本身并不会直接调用CMA模块的接口，而是通过DMA mapping framework来间接使用CMA的服务。
设计之初，CMA area的概念是全局的，通过**内核配置参数**和**命令行参数**，内核可以定位到Global CMA area在内存中的起始地址和大小。并在初始化的时候，调用dma_contiguous_reserve函数，将指定的memory region保留给Global CMA area使用。
后续又引入了per device cma的概念，为了解决个别驱动独占cma的情况。
Global CMA area给大家共享，而per device CMA可以给指定的一个或者几个驱动使用。
此时，命令行参数不能完全描述cma的属性，因此引入了device tree中的reserved memory node的概念。
为了保持兼容性，内核仍然支持CMA的command line参数。

## 3.3 CMA area的管理
在CMA模块中，struct cma数据结构用来抽象一个CMA area：
```C
struct cma {
	unsigned long   base_pfn;
	unsigned long   count;
	unsigned long   *bitmap;
	unsigned int order_per_bit; /* Order of pages represented by one bit */
	spinlock_t	lock;
	char name[CMA_MAX_NAME];
	bool reserve_pages_on_error;
};
```
![](image-5.png)
base_pfn：表示CMA区域的基础页帧号（Page Frame Number），物理内存被分割成多个固定大小的页，每个页都有唯一的页帧号来标识，base_pfn指定了CMA区域起始页的页帧号。
count：表示CMA区域中总的页帧数，从而可以计算出这个区域的总大小（count * PAGE_SIZE，其中PAGE_SIZE是每页的大小）。
bitmap: 是一个指向位图的指针，用于跟踪CMA区域内哪些页是空闲的，哪些页已被分配。
order_per_bit: 指定了位图中每一位代表的页面数量。例，如果是3，那么位图中的每一位就代表2^3=8个页面。
lock: 自旋锁，用于保护CMA区域的并发访问。
name: 字符数组，用于存储CMA区域的名称
reserve_pages_on_error: 指示当CMA区域无法满足分配请求时是否应该保留。如果设置为true，则即使分配失败，已分配的页面也不会被释放回系统。

说明：
1. cma模块使用bitmap来管理其内存的分配，0表示free，1表示已经分配。
2. 内存管理的单位和struct cma中的order_per_bit成员相关，如果order_per_bit等于0，表示按照1个page来分配和释放，如果order_per_bit等于1，表示按照2个page组成的block来分配和释放，以此类推。
3. struct cma中的bitmap成员就是管理该cma area内存的bit map。
4. count成员说明了该cma area内存有多少个page。它和order_per_bit一起决定了bitmap指针指向内存的大小。
5. base_pfn定义了该CMA area的起始page frame number，base_pfn和count一起定义了该CMA area在内存中的位置。

CMA模块需要管理若干个CMA area，有gloal的，有per device的，代码如下：
```c
//mm/cma.c
struct cma cma_areas[MAX_CMA_AREAS];
```
每一个struct cma抽象了一个CMA area，标识了一个物理地址连续的memory area。
调用cma_alloc分配的连续内存就是从CMA area中获得的。
```C
struct page *cma_alloc(struct cma *cma, unsigned long count,
		       unsigned int align, bool no_warn);
```
具体有多少个CMA area是编译时决定了，具体要配置多少个CMA area是和系统设计相关。
可以为特定的驱动准备一个CMA area，也可以只建立一个通用的CMA area，供多个驱动使用。

## 3.4 CMA的配置过程
### 3.4.1 dts配置
上文中说明了cma area的分配，但并没有说明如何配置cma area。
配置CMA内存区有两种方法，一种是通过dts的reserved memory，另外一种是通过command line参数和内核配置参数。
```c
	reserved-memory {
		#address-cells = <2>;
		#size-cells = <2>;
		ranges;

		/* global autoconfigured region for contiguous allocations */
		linux,cma {
			compatible = "shared-dma-pool";
			reusable;
			/* 640 MiB */
			size = <0 0x28000000>;
			/*  1024 - 128 MiB, our minimum RAM config will be 1024 MiB */
			alloc-ranges = <0 0x40000000 0 0x78000000>;
			linux,cma-default;
		};
	};
```
device tree中可以包含reserved-memory node，在该节点的child node中，可以定义各种保留内存的信息。compatible属性中shared-dma-pool节点是专门用于建立 global CMA area的，而其他的child node都是for per device CMA area的。
linux,cma节点中，size属性指定了global CMA area的大小。alloc-ranges属性指定了global CMA area的起始地址和大小。reusable属性表示该节点中的内存是可以重复使用。
linux,cma-default属性表示该节点中的内存是默认的global CMA area。

Global CMA area通过如下方式进行初始化：
```c
RESERVEDMEM_OF_DECLARE(cma, "shared-dma-pool", rmem_cma_setup);
```
具体的setup过程倒是比较简单，从device tree中可以获取该memory range的起始地址和大小，调用cma_init_reserved_mem函数即可以注册一个CMA area。

其中需要说明的是：
CMA对应的reserved memory节点必须有reusable属性，不能有no-map的属性。
**reusable属性**：即在驱动不使用这些内存的时候，OS可以使用这些内存，而当驱动从这个CMA area分配memory的时候，OS可以reclaim这些内存，让驱动可以使用它。
**no-map属性**: 和地址映射相关，如果没有no-map属性，OS会为这段memory创建地址映射，像其他普通内存一样。
有no-map属性的memory往往是专用于某个设备驱动，在驱动中会进行io remap，如果OS已经对这段地址进行了mapping，而驱动又一次mapping，**相当于两段虚拟地址对应同一段物理地址**，这样就有不同的虚拟地址mapping到同一个物理地址上去，在某些ARCH上，会造成不可预知的后果。而CMA这个场景，reserved memory必须要mapping好，这样才能用于其他内存分配场景，例如page cache。

per device CMA area的注册过程和各自具体的驱动相关，但是最终会dma_declare_contiguous这个接口函数，为一个指定的设备而注册CMA area。

### 3.4.2 命令行配置
```C
	cma=nn[MG]@[start[MG][-end[MG]]]
			[KNL,CMA,EARLY]
			Sets the size of kernel global memory area for
			contiguous memory allocations and optionally the
			placement constraint by the physical address range of
			memory allocations. A value of 0 disables CMA
			altogether. For more information, see
			kernel/dma/contiguous.c
```
通过命令行参数也可以建立cma area。
可以通过cma=nn[MG]@[start[MG][-end[MG]]]这样命令行参数来指明Global CMA area在整个物理内存中的位置。
在初始化过程中，内核会解析这些命令行参数，获取CMA area的位置（起始地址，大小），并调用cma_declare_contiguous接口函数向CMA模块进行注册（当然，和device tree传参类似，最终也是调用cma_init_reserved_mem接口函数）。
除了命令行参数，通过内核配置（CMA_SIZE_MBYTES和CMA_SIZE_PERCENTAGE）也可以确定CMA area的参数。

# 4.CMA是如何工作的？
## 4.1 migrate types 和pageblocks
想要了解CMA是如何运作的，需要知道migrate types和pageblocks的概念。
* migrate types 迁移类型
迁移类型: 通过将内存页面根据其可移动性进行分类，来减少内存碎片并提高内存分配的效率。
具体来说，Linux内核中主要有以下几种迁移类型：
UNMOVABLE（不可移动）：这类页面不能被移动，因为它们被硬件或系统关键部分所使用，如内核代码和数据、硬件DMA缓冲区等。
RECLAIMABLE（可回收）：这类页面虽然不能直接移动，但可以在内存压力较大时被回收，用于其他目的。
MOVABLE（可移动）：这类页面可以随时被移动，因为它们不包含固定位置的数据或代码。当系统需要优化内存布局或解决内存碎片问题时，这些页面可以被重新定位到更合适的位置。

* pageblocks（页块）
通常指的是一种内存分配和管理的基本单位，它的大小通常是多个物理页面（pages）的组合。
在一般意义上，页块可以被视为一种比单个页面更大的内存管理单元，用于处理内存分配、回收、碎片整理等任务。

## 4.2 请求过程
当从伙伴系统请求内存的时候，需提供了gfp_mask的参数，在CMA这个场景，它用来指定请求页面的迁移类型(migrate type), 并将类型标记为：MIGRATE_MOVABLE类型，说明该页面上的数据是可以迁移的。这意味着可以分配一个新的page，copy数据到这个新page上去，释放这个page(完成这样的操作对系统没有任何的影响)。

举例：
内核中的data section，其对应的page不是是movable的，因为一旦移动数据，那么内核模块就无法访问那些页面上的全局变量了,而对于page cache这样的页面，其实是可以搬移的，只要让指针指向新的page就可以了。

伙伴系统不会跟踪每一个page frame的迁移类型，实际上它是按照pageblock为单位进行管理的，memory zone中会有一个bitmap，指明该zone中每一个pageblock的migrate type。在处理内存分配请求的时候，一般会首先从和请求相同migrate type（gfp_mask）的pageblocks中分配页面。如果分配不成功，不同migrate type的pageblocks中也会考虑，甚至可能改变pageblock的migrate type。
这意味着一个non-movable页面请求也可以从migrate type是movable的pageblock中分配。这一点CMA是不能接受的，所以我们引入了一个新的migrate type：MIGRATE_CMA。这种迁移类型具有一个重要性质：只有可移动的页面可以从MIGRATE_CMA的pageblock中分配。

## 4.3 初始化CMA area
```c
static int __init cma_activate_area(struct cma *cma)
{
    int bitmap_size = BITS_TO_LONGS(cma_bitmap_maxno(cma)) * sizeof(long);
    unsigned long base_pfn = cma->base_pfn, pfn = base_pfn;
    unsigned i = cma->count >> pageblock_order; //（1）
    struct zone *zone; 

    cma->bitmap = kzalloc(bitmap_size, GFP_KERNEL); //分配内存

    zone = page_zone(pfn_to_page(pfn)); //找到page对应的memory zone

    do {//（2）
        unsigned j;

        base_pfn = pfn;
        for (j = pageblock_nr_pages; j; --j, pfn++) {//（3）
            if (page_zone(pfn_to_page(pfn)) != zone)
                goto err;
        }
        init_cma_reserved_pageblock(pfn_to_page(base_pfn));//（4）
    } while (--i);

    mutex_init(&cma->lock);

    return 0;

err:
    kfree(cma->bitmap);
    cma->count = 0;
    return -EINVAL;
}
```
（1）CMA area有一个bitmap来管理各个page的状态，这里bitmap_size给出了bitmap需要多少的内存。i变量表示该CMA area有多少个pageblock。

（2）遍历该CMA area中的所有的pageblock。

（3）确保CMA area中的所有page都是在一个memory zone内，同时累加了pfn，从而得到下一个pageblock的初始page frame number。

（4）将该pageblock导入到伙伴系统，并且将migrate type设定为MIGRATE_CMA。

## 4.4 分配连续内存
cma_alloc用来从指定的CMA area上分配count个连续的page frame，按照align对齐。
具体的代码就不再分析了，实际上就是从bitmap上搜索free page的过程，一旦搜索到，就调用alloc_contig_range向伙伴系统申请内存。
需要注意的是，CMA内存分配过程是一个比较“重”的操作，可能涉及页面迁移、页面回收等操作，因此不适合用于atomic context。

## 4.5 释放连续内存
分配连续内存的逆过程，除了bitmap的操作之外，最重要的就是调用free_contig_range，将指定的pages返回伙伴系统。




# 参考文档
1.[CMA模块学习笔记](http://www.wowotech.net/memory_management/cma.html)
