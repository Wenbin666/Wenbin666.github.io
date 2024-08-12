---
title: Linux 文件系统简述
date: 2023-08-12 20:05:15
tags: filesystem
categories: 
 - OS
---
# 1. 引言
Linux 文件系统是 Linux 操作系统中不可或缺的核心组件，它对于数据的存储、组织、访问控制、系统稳定性和可靠性等方面都具有至关重要的作用，字符设备和块设备也都良好的体现了 “一切皆是文件” 的设计思想，掌握linux 文件系统、设备文件的知识体系是驱动开发工程师的基础。

# 2. Linux文件系统的目录结构
**/bin** : 包含基本命令，如ls、cp、mkdir等。
**/sbin** : 包含系统命令，如modprobe、ifconfig等，为系统管理相关命令。
**/dev** : 设备文件储存目录，应用程序通过对这些文件的读写和控制来访问实际的设备。
**/etc** : 系统配置文件，如用户账号密码配置文件等。
**/lib**: 存放系统库文件。
**/mnt**: 存放挂载存储设备的挂载目录。
**/opt**: 可选目录，有些软件包会被安装在此目录。
**/proc** : 系统启动后，进程和内核信息存放在这里，proc不是真正的文件系统，它存在于内存之中。
**/tmp** : 存放用户程序产生的临时文件。
**/usr** : 系统存放程序的目录，比如用户命令，用户库等。
**/var** : 存放变化的内容，如log等。
**/sys**:内核的sysfs被映射到此目录。Linux设备驱动模型中的总线bus、驱动和设备都可以在sysfs文件系统找到对应的节点。

# 3. 文件系统与设备驱动的关系
![文件系统与设备驱动的关系](image.png)
* 应用程序和VFS直接的接口是系统调用(创建、打开、读写和关闭等)。
* VFS和“文件系统以及设备驱动“直接的接口是file_operations结构体成员函数。
* 字符设备的上层没有类似于磁盘ext2等文件系统，所以字符设备的file_operations成员函数直接由设备驱动提供。
* 块设备由两种访问方式：
    + 1. 通过访问/dev/sda设备文件的方式，底层是通过具体的def_blk_fops的file_operations去访问。
    + 2. 直接通过文件系统来访问，file_operation的实现位于文件系统内，文件系统会把对文件的读写转化为对块设备原始扇区的读写。

# 4. 关键数据结构
## 4.1 file
```c
struct file {
	union {
		/* fput() uses task work when closing and freeing file (default). */
		struct callback_head 	f_task_work;
		/* fput() must use workqueue (most kernel threads). */
		struct llist_node	f_llist;
		unsigned int 		f_iocb_flags;
	};

	/*
	 * Protects f_ep, f_flags.
	 * Must not be taken from IRQ context.
	 */
	spinlock_t		f_lock;
	fmode_t			f_mode;
	atomic_long_t		f_count;
	struct mutex		f_pos_lock;
	loff_t			f_pos;
	unsigned int		f_flags;
	struct fown_struct	f_owner;
	const struct cred	*f_cred;
	struct file_ra_state	f_ra;
	struct path		f_path;
	struct inode		*f_inode;	/* cached value */
	const struct file_operations	*f_op;

	u64			f_version;
#ifdef CONFIG_SECURITY
	void			*f_security;
#endif
	/* needed for tty driver, and maybe others */
	void			*private_data;

#ifdef CONFIG_EPOLL
	/* Used by fs/eventpoll.c to link all the hooks to this file */
	struct hlist_head	*f_ep;
#endif /* #ifdef CONFIG_EPOLL */
	struct address_space	*f_mapping;
	errseq_t		f_wb_err;
	errseq_t		f_sb_err; /* for syncfs */
} __randomize_layout
  __attribute__((aligned(4)));	/* lest something weird decides that 2 is OK */
```
file结构代表一个打开的文件，系统中每打开一个文件在内核中都有一个关联的stuct file.
* 最开始使用了union联合体，union 的大小等于其最大成员的大小，此联合体允许file使用多种不同的方式来处理文件关闭和释放时的回调。
* f_task_work ：用于默认情况，当关闭和释放文件时，使用任务工作队列（task work）来处理
* f_llist： 对于大多数内核线程，关闭文件时必须使用工作队列（workqueue）来处理
* f_iocb_flags：存储与 I/O 操作相关的状态或配置标志
* f_lock: 用于保护 f_ep 和 f_flags 等字段的访问，确保在多线程环境下的一致性
* f_mode: 表示文件的打开模式（只读、只写、读写等）
* f_count: 引用计数，表示打开该文件的文件描述符的数量。当 f_count 降为0时，文件将被关闭
* f_pos_lock：互斥锁，用于保护文件当前读写位置 f_pos 的访问
* f_pos ： 当前读写的位置
* f_flags：包含了文件的各种状态标志，如是否已同步、是否异步等
* f_owner：文件的所有者信息
* f_cred：f_cred 指向包含进程凭证（如UID、GID等）的结构体
* f_ra：读前缓存（read-ahead）的状态信息，以提高顺序读取的效率
* f_path：文件的路径信息
* f_inode ：缓存文件的inode结构体指针，用于快速访问文件的元数据
* f_op：**file_operations, 和文件关联的操作**
* f_version: 版本信息，用于跟踪文件变更
* f_security： secure相关内容
* private_data：允许文件系统的实现或驱动程序存储与文件相关的私有数据
* f_ep：当内核配置了epoll支持时，此字段用于将文件链接到epoll实例，以支持高效的I/O事件通知
* f_mapping：指向文件的地址空间对象，用于管理文件的内存映射
* f_wb_err：用于同步文件写操作的错误检测
* f_sb_err: 用于同步文件系统操作时的错误检测
__randomize_layout用于随机成员存储，aligned用于字节对齐

## 4.2 inode
```C
struct inode {
	umode_t			i_mode;
	unsigned short		i_opflags;
	kuid_t			i_uid;
	kgid_t			i_gid;
	unsigned int		i_flags;
	const struct inode_operations	*i_op;
	struct super_block	*i_sb;
	struct address_space	*i_mapping;
	unsigned long		i_ino;
	union {
		const unsigned int i_nlink;
		unsigned int __i_nlink;
	};
	dev_t			i_rdev;
	loff_t			i_size;
	struct timespec64	__i_atime;
	struct timespec64	__i_mtime;
	struct timespec64	__i_ctime; /* use inode_*_ctime accessors! */
	spinlock_t		i_lock;	/* i_blocks, i_bytes, maybe i_size */
	unsigned short          i_bytes;
	u8			i_blkbits;
	enum rw_hint		i_write_hint;
	blkcnt_t		i_blocks;

	/* Misc */
	unsigned long		i_state;
	struct rw_semaphore	i_rwsem;

	unsigned long		dirtied_when;	/* jiffies of first dirtying */
	unsigned long		dirtied_time_when;

	struct hlist_node	i_hash;
	struct list_head	i_io_list;	/* backing dev IO list */
	struct list_head	i_lru;		/* inode LRU list */
	struct list_head	i_sb_list;
	struct list_head	i_wb_list;	/* backing dev writeback list */
	union {
		struct hlist_head	i_dentry;
		struct rcu_head		i_rcu;
	};
	atomic64_t		i_version;
	atomic64_t		i_sequence; /* see futex */
	atomic_t		i_count;
	atomic_t		i_dio_count;
	atomic_t		i_writecount;
	union {
		const struct file_operations	*i_fop;	/* former ->i_op->default_file_ops */
		void (*free_inode)(struct inode *);
	};
	struct file_lock_context	*i_flctx;
	struct address_space	i_data;
	struct list_head	i_devices;
	union {
		struct pipe_inode_info	*i_pipe;
		struct cdev		*i_cdev;
		char			*i_link;
		unsigned		i_dir_seq;
	};

	__u32			i_generation;
	void			*i_private; /* fs or device private pointer */
} __randomize_layout;
```
inode是管理文件系统的基本单位，也是文件系统连接任何子目录、文件的桥梁。
* i_mode：文件的类型和权限，包含了文件类型（如普通文件、目录、字符设备等）和权限信息（如读、写、执行权限）。
* i_opflags：操作的标志位，用于控制inode的某些特定行为。
* i_uid 和 i_gid：分别表示文件所有者的用户ID和组ID。
* i_flags：inode的标志位，用于控制inode的特定行为，如文件是否被压缩、加密等。
* i_op：指向inode操作函数表的指针。
* i_sb：指向超级块（super_block）的指针，超级块包含了文件系统的全局信息，如文件节点数、块大小等。
* i_mapping：地址空间管理文件的页缓存。
* i_ino：inode号，每个inode都有一个唯一的inode号，用于唯一标识文件系统中的一个节点。
* i_nlink：硬链接数，union 类型中的一个成员。表示有多少文件名指向这个inode。
* i_rdev：如果inode表示一个设备文件，这个字段表示设备的类型和设备号。
* i_size：文件大小，表示文件的数据部分占用的字节数。
* i_atime、i_mtime、i_ctime：分别表示文件的访问时间、修改时间和状态改变时间。
* i_lock：自旋锁，用于保护inode中的某些字段，如i_blocks、i_bytes等。
* i_blocks：文件占用的块数，用于计算文件占用的磁盘空间。
* i_state：状态标志，用于表示inode的当前状态，如是否被锁定、是否脏等。
* i_rwsem：读写信号量，保护inode，防止并发访问。
* dirtied_when：inode首次被标记为脏时的jiffies值
* i_hash、i_io_list、i_lru、i_sb_list、i_wb_list：分别用于将inode插入到哈希表、IO列表、LRU列表、超级块列表和写回列表中。
* i_dentry：表示inode的目录项，用于快速查找文件名对应的inode
* i_rcu：表示Read-Copy Update头部，用于RCU机制的内存回收。
* i_version、i_sequence、i_count、i_dio_count、i_writecount：原子类型的计数器，用于管理inode的版本号、序列号、引用计数等。
* i_fop 或 free_inode：表示inode的文件操作函数表或释放inode的函数指针。
* i_flctx：指向文件锁上下文的指针，用于管理文件锁。
* i_data：inode的数据地址空间，管理文件的页缓存。
* i_devices：与inode关联的设备列表。
* i_pipe、i_cdev、i_link、i_dir_seq：分别表示inode是否为管道、字符设备、符号链接或目录的序列号。
* i_generation: inode的geration号，主要用于缓存一致性、文件同步和复制过程中识别inode的变更。
* i_private: 文件系统或设备驱动程序存储与inode相关的私有数据。

# 5. devfs 简介
devfs,设备文件系统由linux2.4版本引入(2.6版本抛弃)，它的出现使得设备驱动程序能自主的管理自己的设备文件。
devfs的优点：
1. 可以通过程序在设备初始化时在/dev目录创建设备文件，卸载时将其删除。
2. 设备驱动程序可以指定设备名、所有者和权限位，用户空间仍可以修改所有者和权限位。
3. 不再需要为设备分配主设备号及其处理次设备号，
## 5.1 设备文件的创建和撤销API
### 5.1.1 devfs_mk_dir
创建设备目录
```C
devfs_handle_t devfs_mk_dir (devfs_handle_t dir, const char *name, void *info);
```
尝试在指定的父目录 dir 下创建一个名为 name 的新目录，成功，返回新目录的句柄，失败，返回错误指示。
dir：指向要在其中创建新目录的父目录。
name：字符串的指针，该字符串包含了要创建的目录的名称。
info：可能被用来存储与新目录相关联的元数据、权限信息、设备标识符或其他任何特定于应用的数据。

### 5.1.2 devfs_register
创建设备文件
```C
devfs_handle_t devfs_register (devfs_handle_t dir, const char *name,
				      unsigned int flags,
				      unsigned int major, unsigned int minor,
				      umode_t mode, void *ops, void *info);
```
用于在设备文件系统中注册一个新设备节点的函数
* dir：指定新设备节点应该注册到的父目录的句柄。
* name：新设备节点在父目录中的名称。
* flags：注册标志，用于控制注册过程的某些方面。
* major, minor：指定了设备的主设备号和次设备号。
* mode：指定了设备节点的文件模式和权限。
* ops：设备操作函数的结构体。定义了如何对设备进行读写、控制等操作。
* info：这是一个指向任意类型数据的指针，用于传递与新设备节点相关的额外信息。

### 5.1.3 devfs_unregister
撤销设备文件
```c
void devfs_unregister (devfs_handle_t de);
```
用于从设备文件系统（Device File System, devfs）中注销（或删除）一个已注册的设备节点的函数
* de：唯一地标识了设备文件系统中要注销的设备节点

# 6. udev用户空间设备管理
尽管devfs有一些优点，但是在linux2.6的内核中，它被认为是过时的方法，udev取代了它。
主要原因如下：
1. **用户态实现**：devfs所做的工作被确信可以在用户态完成，用户态的程序通常比内核态的程序更容易调试和更新。
2. **质量和稳定性问题**：devfs存在一些可修复和无法修复的bug。其稳定性和可靠性受到了质疑。
3. **维护放弃**：由于上述质量和稳定性问题，维护者和作者停止了对代码的维护工作。
4. **设计理念差异**：udev和devfs在设计理念上存在差异。udev完全在用户态工作，它利用设备加入或移除时内核所发送的热插拔事件来管理设备文件。这种方式使得udev能够更灵活地处理设备文件的创建、删除和权限设置等操作。而devfs则试图在内核态完成这些工作，这在实践中被证明是复杂且容易出错的。
5. **设备管理和驱动加载**：udev可以在设备被发现时加载相应的驱动模块，而devfs是在设备文件被访问时。这种设计理念使得udev能够更高效地管理设备文件和驱动模块，避免了不必要的资源浪费和潜在的冲突。
6. **动态设备文件管理**：udev能够根据系统中硬件设备的状态动态地更新设备文件，进行设备文件的创建和删除等操作。这使得在/dev目录下只包含系统中真正存在的设备文件，避免了因设备文件与实际硬件设备不匹配而导致的问题。

## 6.1 udev netlink 匹配机制
udev的netlink匹配机制是udev系统中一个关键的部分，它涉及到udev如何接收内核发送的设备事件（uevent），并根据这些事件来执行相应的操作，如创建设备节点、设置权限等。
具体流程如下：

1. 内核发送 uevent
当设备被插入或移除时，内核会检测到这一变化，并通过netlink套接字发送一个uevent到用户空间。这个uevent包含了设备的详细信息，如设备类型、路径、子系统等。
2. udevd 监听 uevent
udevd 是 udev 的守护进程，它在系统启动时运行，并持续监听来自内核的uevent。当udevd 接收到一个uevent时，它会解析这个事件，并准备根据 udev 规则来执行相应的操作。
3. 匹配 udev 规则
udev规则文件存放在 /etc/udev/rules.d/目录下，这些文件定义了如何根据设备的属性来识别设备，并确定应该采取何种操作。
udevd 会遍历这些规则文件，查找与接收到的 uevent 相匹配的规则。匹配过程通常基于设备的属性，如设备名称、子系统、总线类型等。其中规则文件中的规则以行为单位，每行代表一个规则。规则分为匹配键和赋值键两部分，匹配键用于识别设备，赋值键用于指定对设备的操作。
4. 执行操作
一旦找到匹配的规则，udevd 就会执行规则中定义的操作。这些操作可能包括创建设备文件、设置设备文件的权限和所有者/组、加载驱动程序、执行自定义脚本等。udevd 还会根据规则中的设置，为设备文件创建符号链接、设置环境变量等，以便于用户和其他程序访问设备。
5. 通知系统
设备准备好后，udevd 会发送一个通知给系统，告诉其他程序设备已经可以使用了。这样，用户就可以通过相应的设备文件来访问设备了。

使用udev的netlink机制进行设备事件的传递和匹配，使得 udev 能够高效地处理大量的设备事件，特别是在设备频繁插拔的场合下，udev 规则文件的灵活性和可扩展性，使得系统管理员可以根据需要自定义设备的管理策略，满足不同的应用场景。

# 7. sysfs文件系统与linux设备模型
Linux2.6以后的内核引入了sysfs文件系统，sysfs被看成是与proc和devfs同类别的文件系统，该文件系统是一个虚拟的文件系统，它可以产生一个包括所有系统硬件的层级视图，与提供进程和状态信息的proc文件系统十分类似。
* sysfs把连接在系统上的**设备**和**总线**组成了一个分级文件，它们可以由用户空间存取，向用户空间导出内核数据结构以及它们的属性。
* sysfs的一个目的就是展示设备驱动模型中各个组件的层次关系。
其顶级目录包含block、bus、dev、devices、class、fs、kernel、power和firmware等。

* block：包含系统中所有的块设备信息。块设备是以块为单位进行数据传输的设备。
* bus：包含系统中注册的所有总线类型。总线是计算机中连接各个设备的一种物理通道。
* dev：以原始方式（无层次结构）包含已注册的设备节点，提供了一种快速访问设备信息的方式，而不需要遍历整个设备树。
* devices： 这是系统中所有设备存放的目录，该目录下的设备按照总线类型组织成层次结构，每种设备都挂在某种总线之下。这个目录直接反映了系统中设备的实际连接情况，是用户空间程序获取设备信息的主要途径。
* class: 包含系统中的**设备类**信息。设备类是一组具有共同属性的设备，它们可能由不同的驱动程序控制，但具有相似的功能和操作方式。在class目录下，每个设备类都有一个对应的目录，其中包含了属于该类的所有设备信息。这种方式有助于用户空间程序根据设备类来管理设备。
* fs: 描述系统中所有文件系统，包括文件系统本身和按文件系统分类存放的已挂载点。它更多地与文件系统的管理和挂载状态相关，而不是直接反映设备或驱动的信息。
* kernel: 包含与内核相关的各种信息和属性，通常包括一些内核模块、参数、日志等信息。
* power: 包含系统中电源管理的相关信息和属性，如控制电源状态（如休眠、唤醒等）的接口和文件。
* firmware：描述了内核中的固件信息。

可以看到不同目录下可能包含相同的内容，通过一些软连接进行覆盖，整体结构如下图：
![Linux设备模型](image-1.png)

说明：总线、驱动和设备最终都会落实为sysfs中的一个目录，进一步追踪代码会发现，它们其实都可以认为是kobject的派生类，kobject可以看作是所有总线、设备和驱动的抽象基类，一个kobject对应sysfs中的一个目录。

# 8.udev规则文件
## 8.1 udev规则文件简介
上文已经说明的udev规则文件的内容，此处进行详细说明
* udev的规则文件以行为单位，以”#“为注释符号。每一行代表一个规则。
* 每个规则分成一个或多个匹配部分和赋值部分，匹配部分用专用的匹配关键字，赋值部分用相应的赋值关键字。
* 匹配关键字：
    + ACTION(行为)、KERNEL(内核设备名)、BUS(匹配总线)、SUBSYSTEM、ATTR。
* 赋值关键字：
    + NAME(创建设备名)、SYSMLINK(符号链接)、OWNER(所有者)、GROUP(设备组)、IMPORT(调用外部程序)、MODE(访问权限)

举例：
```C
SUBSYSTEM=="net", ACTION=="add", DRIVERS=="?*", ATTR{address}=="08:00:27:35:be:ff",
    ATTR{dev_id}=="0x0", ATTR{type}=="1", KERNEL=="eth", NAME="eth1"
```
解释：当系统中出现的新硬件属于net子系统范畴，系统对硬件采取的动作是”add“这个硬件，且这个硬件的”address“属性信息等于
”08:00:27:35:be:ff“，dev_id属性等于0x0，type属性为1，此时对这个硬件在udev层次施行的动作是创建/dev/eth1

## 8.2 udevadm工作
udevadm是一个在Linux系统中用于管理和控制udev设备的命令行工具。
udevadm工具的主要功能：
* 查询设备信息：udevadm info命令可以查询udev数据库中关于设备的详细信息，包括设备名称、厂商信息、设备类型、驱动程序等。
* 监控设备事件：udevadm monitor命令可以监控内核事件和udev事件，帮助用户了解设备连接、断开等事件的发生情况。
* 模拟和测试udev事件：udevadm test命令可以模拟一次udev事件，以测试udev规则是否正确。
* 重新触发udev事件：udevadm trigger命令可以重新触发udev事件，以确保udev根据最新的规则和设备状态更新设备文件。
* 等待所有udev事件处理完成：udevadm settle命令会阻塞等待，直到所有待处理的udev事件都被处理完成。

# 9. 总结
Linux 文件系统和 udev 都是 Linux 操作系统中不可或缺的重要组成部分。
文件系统负责组织和存储数据，而 udev 则负责管理和控制硬件设备，两者共同协作，为 Linux 系统提供了强大而灵活的文件和设备管理能力。