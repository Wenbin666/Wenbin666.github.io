---
title: Linux字符设备驱动
date: 2021-07-10 20:37:00
tags: cdev
categories: 
 - kernel
---

# 1. 前言
字符设备驱动是Linux系统中最基本的一类设备驱动，用于管理和控制那些只能以字节或字符为单位进行读写操作的设备。
**字符设备的特点**：
* 面向流的数据传输：字符设备按字节或字符序列进行数据传输，不能随机访问设备内存中的任意位置。
* 顺序访问：数据读取必须按照先后顺序进行。
* 常见设备：包括鼠标、键盘、串口、控制台和LED设备等。

# 2. 关键数据结构
## 2.1 cdev
```C
struct cdev {
	struct kobject kobj;   // 内嵌的kobject 对象，用于实现对象间的层次关系和生命周期管理
	struct module *owner;  // 所属模块
	const struct file_operations *ops; //文件操作结构
 	struct list_head list; //链表头，用于将字符设备链接到内核的字符设备列表中,方便遍历与管理
	dev_t dev;  //设备号
	unsigned int count; //记录当前cdev关联的次设备号数量
} __randomize_layout;   //编译器指令，用于随机化结构体的布局，以增加安全性
```
cdev为描述字符设备的结构体。
## 2.2 file_operations
```C
struct file_operations {
	struct module *owner;
	fop_flags_t fop_flags;
	loff_t (*llseek) (struct file *, loff_t, int);
	ssize_t (*read) (struct file *, char __user *, size_t, loff_t *);
	ssize_t (*write) (struct file *, const char __user *, size_t, loff_t *);
	ssize_t (*read_iter) (struct kiocb *, struct iov_iter *);
	ssize_t (*write_iter) (struct kiocb *, struct iov_iter *);
	int (*iopoll)(struct kiocb *kiocb, struct io_comp_batch *,
			unsigned int flags);
	int (*iterate_shared) (struct file *, struct dir_context *);
	__poll_t (*poll) (struct file *, struct poll_table_struct *);
	long (*unlocked_ioctl) (struct file *, unsigned int, unsigned long);
	long (*compat_ioctl) (struct file *, unsigned int, unsigned long);
	int (*mmap) (struct file *, struct vm_area_struct *);
	int (*open) (struct inode *, struct file *);
	int (*flush) (struct file *, fl_owner_t id);
	int (*release) (struct inode *, struct file *);
	int (*fsync) (struct file *, loff_t, loff_t, int datasync);
	int (*fasync) (int, struct file *, int);
	int (*lock) (struct file *, int, struct file_lock *);
	unsigned long (*get_unmapped_area)(struct file *, unsigned long, unsigned long, unsigned long, unsigned long);
	int (*check_flags)(int);
	int (*flock) (struct file *, int, struct file_lock *);
	ssize_t (*splice_write)(struct pipe_inode_info *, struct file *, loff_t *, size_t, unsigned int);
	ssize_t (*splice_read)(struct file *, loff_t *, struct pipe_inode_info *, size_t, unsigned int);
	void (*splice_eof)(struct file *file);
	int (*setlease)(struct file *, int, struct file_lease **, void **);
	long (*fallocate)(struct file *file, int mode, loff_t offset,
			  loff_t len);
	void (*show_fdinfo)(struct seq_file *m, struct file *f);
#ifndef CONFIG_MMU
	unsigned (*mmap_capabilities)(struct file *);
#endif
	ssize_t (*copy_file_range)(struct file *, loff_t, struct file *,
			loff_t, size_t, unsigned int);
	loff_t (*remap_file_range)(struct file *file_in, loff_t pos_in,
				   struct file *file_out, loff_t pos_out,
				   loff_t len, unsigned int remap_flags);
	int (*fadvise)(struct file *, loff_t, loff_t, int);
	int (*uring_cmd)(struct io_uring_cmd *ioucmd, unsigned int issue_flags);
	int (*uring_cmd_iopoll)(struct io_uring_cmd *, struct io_comp_batch *,
				unsigned int poll_flags);
} __randomize_layout;
```
**fop_flags** : 标志位字段，用于表示该结构体的某些特殊属性或行为
**llseek** :修改一个文件的当前读写位置，并将新位置返回，在出错时返回负值
**read** : 从设备读数据，成功时返回读取字节数，否则返回负值，与用户空间read，fread对应
**write**: 向设备发送数据，成功时返回写入的字节数,与用户空间write，fwrite对应，返回0代表eof
**ead_iter 和 write_iter**： 是read和write的迭代版本，得益于iov_iter结构，可以更高效的完成读写
**iopoll**：异步I/O操作的轮询，允许在 I/O 操作完成时通知内核，而不需要内核主动轮询
**iterate_shared**：用于目录的迭代，特别是当需要遍历目录中的文件时
**poll**：用于询问设备是否可以被非阻塞地立即读写，当询问条件未触发时，用户空间select和poll调用会导致进程阻塞
**unlocked_ioctl**：提供设备相关控制命令的实现(非读写)，与用户空间的fcntl、ioctl相对应
**compat_ioctl**：unlocked_ioctl 是现代驱动应使用的版本，compat_ioctl用于处理与旧版二进制兼容的ioctl命令
**mmap**：将设备内存映射到进程的虚拟地址空间中
**open**：当用户空间调用open打开文件时，设备驱动会调用这个open函数，与release函数相对应
**flush**： 内核缓冲区刷新，将缓冲区中的数据写入设备
**fsync**: fsync 用于同步文件的状态到磁盘
**fasync**: fasync 用于设置文件的异步通知机制
**lock**: 文件锁操作相关，防止资源竞争
**get_unmapped_area**：用于请求在进程的地址空间中为设备内存映射分配一个未映射的区域
**check_flags**： 检查标志状态
**flock**：用户设置文件锁
**splice_write、splice_read、splice_eof**：用于高效地在管道或文件之间进行数据传输，而无需复制数据
**setlease**：设置文件的租赁（lease），防止文件在并发访问中被删除的机制
**fallocate**：用于预先分配文件空间，提高写入性能
**show_fdinfo**：用于向调试工具（如 /proc 文件系统）提供有关文件的信息
**mmap_capabilities** ： 用于查询系统对内存映射操作的支持能力
**copy_file_range、remap_file_range**：用于在文件之间高效地复制和重新映射数据块
**fadvise**：允许应用程序向内核提供关于其文件访问模式的建议，以优化性能。
**uring_cmd、uring_cmd_iopoll**：主要用于高性能 I/O 操作，减少系统调用次数和上下文切换
## 2.3 dev_t
```C
typedef u32 __kernel_dev_t;
typedef __kernel_dev_t		dev_t;
```
可以看到设备号是一个u32的类型，那一个数字怎么同时表示主设备和从设备呢？
答： Linux内核提供了MKDEV宏。这个宏接受两个参数：主设备号和次设备号，并将它们组合成一个dev_t类型的值

# 3. 关键API
## 3.1 设备号相关
### 3.1.1 MKDEV
```C
#define MINORBITS	20
#define MKDEV(ma,mi)	(((ma) << MINORBITS) | (mi))
```
通过主设备号和从设备号生成dev_t

### 3.1.2 MAJOR、MINOR
```C
MAJOR(dev_t dev)
MINOR(dev_t dev)
```
从dev_t获得主设备号/从设备号

### 3.1.3 register_chrdev_region
```C
int register_chrdev_region(dev_t from, unsigned count, const char *name);
```
作用：用于已知起始设备号的情况，为字符设备请求一个或多个设备号，并确保这些设备号在当前系统中是唯一的。
**from**：指定请求的设备号范围的起始值
**count**：指定要请求的设备号的数量
**name**：表示设备或者驱动的名称

### 3.1.4 alloc_chrdev_region
```C
int alloc_chrdev_region(dev_t *dev, unsigned baseminor, unsigned count, const char *name);
```
作用：不知道具体设备号的情况下，请求内核为其分配一个或多个可用的设备号
**dev**：这是一个指向 dev_t 类型的指针，用于输出分配到的起始设备号
**baseminor**：指定希望分配的次设备号的起始值
**count**：指定要分配的设备号的数量
**name**：指定请求的设备区域的名称，此这个名称在系统中应唯一，会在 /proc/devices 文件中显示

### 3.1.5 unregister_chrdev_region
```C
void unregister_chrdev_region(dev_t from, unsigned count);
```
释放申请的多个设备号

## 3.2 cdev操作相关
### 3.2.1 cdev_init
```C
void cdev_init(struct cdev *cdev, const struct file_operations *fops);
```
用于初始化cdev的成员，并建立cdev和file_operations直接的连接。
### 3.2.2 cdev_alloc
```C
struct cdev *cdev_alloc(void);
```
用于动态申请一个cdev内存

### 3.2.3 cdev_add
```C
int cdev_add(struct cdev *p, dev_t dev, unsigned count);
```
向系统添加一个cdev设备，完成字符设备的注册。

### 3.2.4 cdev_del
```C
void cdev_del(struct cdev *p);
```
从系统删除一个cdev设备，完成字符设备的注销。

### 3.2.5 cdev_put
```C
void cdev_put(struct cdev *p);
```
减少字符设备对象 cdev 的使用计数，并在计数降至零时释放该对象占用的资源

# 4.框架与源码
![cdev框架](image.png)

## 4.1 字符设备驱动的模块的加载与卸载
注册函数：主要为设备号的申请和cdev的注册
卸载函数：实现设备的注销和已申请设备号的释放
代码框架如下：
```c
#define CUSTOM_DEVICE_NUM 0
#define DEVICE_NUM 1
#device DEVICE_NAME "XXXXXX"
static dev_t global_custom_major = CUSTOM_DEVICE_NUM;
struct xxx_dev_t {
    struct cdev cdev;
    ...
}
static int __init xxx_init(void)
{
    ...
    cdev_init(&xxx_dev_t, &xxx_fops);
    xxx_dev_t.cdev.owner = THIS_MODULE;
    dev_t custom_device_number= MKDEV(global_custom_major, 0);	//	custom device number
    /* device number register*/
    if (global_custom_major) {
        ret = register_chrdev_region(custom_device_number, DEVICE_NUM, DEVICE_NAME);
    } else {
        ret = alloc_chrdev_region(&custom_device_number, 0, DEVICE_NUM, DEVICE_NAME);
        global_custom_major = MAJOR(custom_device_number);
    }
}

static void __exit xxx_exit(void)
{
    cdev_del(&xxx_dev_t);
    unregister_chrdev_region(MKDEV(global_mem_major, 0), DEVICE_NUM);
}

module_init(xxx_init);
module_exit(xxx_exit);
```
## 4.2 file_operation结构体成员函数实现
file_operation结构体的成员函数是字符设备驱动与内核虚拟文件系统的接口，是用户空间对linux进行系统调用的最终落实者。
大多数字符设备驱动会实现read、write和ioctl函数。
```C
ssize_t xxx_read(struct flie *filp, char __user *buf, size_t count, loff_t *f_pos)
{
    ...
    copy_to_user(buf, ..., ...);
    ...
}
ssize_t xxx_write(struct file *filp,  const char __user *buf, size_t count, loff_t *f_pos)
{
    ...
    copy_from_user(buf, ..., ...);
    ...
}
long xxx_ioctl(struct file *filp, unsigned int cmd, unsigned long arg)
{
    ...
    switch (cmd){
        ...
        switch XXX_CMD1:
            ...
            break;
        switch XXX_CMD2:
            ...
            break;
        default:
            return -ENOTTY;
    }
    return 0;
}
```
在设备驱动的读写函数中，filp是文件结构体指针，buf是用户空间内存的地址，该地址在内核空间不宜直接读写，count是要读写的字节数，f_pos是读写的位置相对于文件开头的偏移。
用户空间不能直接访问内核空间的内存，因此借助copy_from_user,copy_to_user完成内核空间到用户空间缓冲区的复制。

## 4.3 用户空间简易代码
```C
#include <stdio.h>
#include <sys/types.h>
#include <sys/stat.h>
#include <fcntl.h>        
#define FILE	"/dev/XXXX"
int main(void)
{
	int fd = -1;
	fd = open(FILE, O_RDWR);
	if (fd < 0){
		printf("open %s error.\n", FILE);
		return -1;
	}
	printf("open %s success..\n", FILE);
	// 读写文件	
	...
	// 关闭文件
	close(fd);	
	return 0;
}
```
## 4.4 mknod命令
mknod 是一个用于创建设备文件的 Linux 命令
```bash
mknod [OPTION] NAME TYPE MAJOR MINOR
```
NAME：要创建的设备文件的名称。
TYPE：设备文件的类型，可以是 b（块设备）或 c（字符设备）。
MAJOR：设备的主设备号。
MINOR：设备的次设备号。

# 5.总结
总的来说，字符设备在系统中扮演着重要的角色，提供了简单而高效的数据传输方式，并且与许多实时数据流和简单交互设备密切相关，也是了解块设备、网络设备等高级内容的基础。