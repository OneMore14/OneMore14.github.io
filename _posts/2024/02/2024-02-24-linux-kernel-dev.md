---
layout: post
title:  "Linux Kernel Development"
date:   2024-02-24 10:43:01 +0800
tags: linux
typora-root-url: ../../../
---
原书: [Linux Kernel Development (3rd Edition)](https://www.doc-developpement-durable.org/file/Projets-informatiques/cours-&-manuels-informatiques/Linux/Linux%20Kernel%20Development,%203rd%20Edition.pdf)

## 3. Process Management

### 3.1 The Process

结束执行但没有被父进程```wait()``` 的是 **Zombie process**，还没有结束执行但父进程已经结束的是 **orphan processes**，孤儿进程会被init进程回收。

僵尸进程会占用process table的一项，用于父进程查询其状态。

### 3.2 Process Descriptor and the Task Structure

进程存储在 ```task list``` 中，是一个双向循环链表。进程的类型是```struct task_struct```，包含打开的文件、地址空间、信号等等信息。

#### 3.2.1 Allocating the Process Descriptor

还有 ```thread_info``` 存储

#### 3.2.2 Storing the Process Descriptor

```/proc/sys/kernel/pid_max``` 可以查看```PID```的最大值，在我的Ubuntu系统上是4194304。

```current``` 宏可以得到当前的```task_struct``` 

#### 3.2.3 Process State

* TASK_RUNNING  要么正在执行，要么在队列中等待执行
* TASK_INTERRUPTIBLE   进程阻塞中(blocked)，等待某个条件成立。这个条件可以是中断、信号量、等待的系统资源等待
* TASK_UNINTERRUPTIBLE   和TASK_INTERRUPTIBLE一样，except that in this case delivering a signal will not wake up the process. This process state is seldom used
* __TASK_TRACED   The process is being traced by another process, such as a debugger, via ```ptrace```.
* __TASK_STOPPED

### 3.3 Process Creation

```fork()```实现可以使用copy-on-write优化。```fork()``` 是通过另一个系统调用 ```clone()```实现的，```clone()``` 会调用 ```do_fork()```。在```do_fork()``` 之后，会先让子进程先执行，期待子进程执行```exec()```，避免父进程写内存导致COW开销。

在Linux中```vfork()``` 和 ```fork()``` 几乎一样，只是不会复制page table。父进程会在子进程运行时阻塞，直到子进程退出或执行```exec()``` ，父进程恢复执行。

### 3.4 The Linux Implementation of Threads

在Linux中，没有单独的thread这个概念，每个线程都有一个自己的```task_struct```，无非是有些数据是共享的。创建线程和创建进程是一样的，只是设置的flag不同。

内核会创建kernel threads用于在后台执行任务，比如*flush* 和 *ksoftirqd* 。内核线程在系统启动后创建。

### 3.5 Process Termination

会释放一系列资源，但之后process不会立马消失，而是变成zombie状态，这样让父进程能查询到子进程的状态，在查询完后才完全消失。如果父进程先退出，会先尝试给当前进程重找一个爹(```find_new_reaper()```)，如果没合适的就交给init进程。



## 4. Process Scheduling

本书主要介绍Linux 2.6的调度器。在Linux5.15中，代码主要在```kernel/sched/fair.c```中



Linux中使用*nice* 来表达进程优先级，数值[-20, 19]，默认为0，nice值越大优先级越低，占的portion越少，nice值可以用```ps -el``` 查看。

另一个优先级维度是*real-time priority*，数值[0, 99]，但不是所有进程都有，只有real-time processes有，可通过```ps -eo state,uid,pid,ppid,rtprio,time,comm``` 查看 ```RTPRIO``` 这一列查询值。

时间片一般都不能太大，设置的比较小，比如10ms。但是Linux CFS(Completely Fair Scheduler)不是给定一个固定值，而是根据系统状态动态分配。

**Fair Scheduling** 

有 $$n$$ 个进程，假设每个进程都分配 $$1/ n$$ 的处理器时间，而时间片又是无限小的。这样在任意一段时间内，每个进程都运行了相同时间，这种情况就是*perfect multitasking* 。但实际上不可能有这样的机制。CFS设置一个*targeted latency*，用于尽量保证一个任务在这个时间内会被执行。例如targeted latency设置成20ms，有两个相同优先级进程，则每个可以执行10ms；如果有20个相同进程，则每个可以执行1ms。当然也能看出来，进程越多，分到的时间越少，调度开销就越大，CFS再设置一个*minimum granularity*，表示分配的最少时间，所以CFS在进程多的时候其实不怎么公平。同时CFS使用nice值会保证两个进程的nice的差才能影响时间占比，比如0和5，10和15都是3:1的时间比例。无所谓nice的绝对值。

**CFS**

CFS用作普通进程(tasks that have no real-time execution constraints)的调度器，没有时间片的概念，但还是会记录进程运行的时间。且调度时按```sched_entity``` 作为单位，一个```task_struct``` 中有 ```sched_entity``` 指针，不同task可能同属一个调度单位。

sched_entity有一个vruntime的u64字段，记录进程实际执行的时间，单位为ns(注意，不是tick数)，但是是经过权重调整后的时间。在理想的公平处理器下，相同优先级进程的vruntime就是不需要额外处理的实际执行时间，且数值相同。

CFS调度的核心是选择vruntime最小的进程(直接取最小就好了，因为nice值高的进程，其实际执行时间大于vruntime，有了vruntime不需要再考虑优先级的问题)，使用红黑树管理可运行的进程，每次选择树中leftmost节点(```__pick_next_entity()```) ，但不用真的遍历，而是缓存该节点，因此 $$O(1)$$ 时间复杂度。一般会由system timer定期更新当前vruntime，还有在进程block或runnable时更新。

CFS向红黑树添加节点一般是在进程变为RUNNABLE或刚被fork出来的时候，函数在```enqueue_entity()``` ，如果插入的节点是leftmost的，就将其记录下来。当进程block或终止的时候调用```dequeue_entity``` 将其移出树。每一个```sched_entity``` 包含一个rb节点(不是指针指向，是自己包含)。

**The Scheduler Entry Point**

Linux调度函数为```schedule()``` ，Linux不止一个调度器(```struct sched_class```)，会按优先级遍历这些调度器，每个调度器有```pick_next_task()``` 方法，选择出一个任务就执行该任务。

**Wait Queues**

对于休眠的进程，要将其放入wait queue。要小心在signal在被放入wait queue之前就已经到了，如果不处理的话就可能一直在wait queue里出不来。

**Preemption and Context Switching**

内核中有一个```need_resched``` flag，指示现在是否应该调度一次，如果为1，则会调用```schedule()```。当内核准备返回user-space(因为一旦返回，理论上说除非用户主动发起会阻塞的系统调用，就会一直运行下去，当然实际上还有timer的中断)或者从中断返回后，会检查这个flag。

Linux的内核进程也是可抢占的，前提是其没有持有任何锁，因此有一个字段```preempt_count``` 在 ```struct thread_info``` 中。初始为0，每获得锁加一，每释放锁减一。

如果中断结束是返回到内核态，也会检查是否可调度，如果可以就调用```schedule()```。此外，内核也可以主动调用```schedule()``` 或调用一些会阻塞的函数间接调用```schedule()```

**Real-Time Scheduling Policies**

SCHED_FIFO 按FIFO执行进程，一直运行知道block或者主动放弃，但可能被更高优先级的实时任务抢占。SCHED_FIFO总是比SCHED_NORMAL先调度。SCHED_RR和SCHED_FIFO是一样的，不同点在于有一个预设置的timeslice，但是这个timeslice只是对同优先级的任务有用。高优先级任务可以立即抢占低优先级任务。低优先级任务不可能抢占高优先级任务，即使高优先级任务运行时间已经超过时间片了。

一般实时优先级的数值为[0, 99] (99是MAX_RT_PRIO - 1)，普通任务的nice值可以直接映射到实时优先级[MAX_RT_PRIO, MAX_RT_PRIO + 40]，MAX_RT_PRIO为100时，nice值为0就是优先级120。

**Scheduler-Related System Calls**

* nice()   Sets a process’s nice value
* sched_setscheduler()   Sets a process’s scheduling policy
* sched_setparam()    Sets a process’s real-time priority
* sched_setaffinity()    Sets a process’s processor affinity

### 4.x Earliest eligible virtual deadline first scheduling

（本节不是书上的内容）

在Linux 6.6内核中，使用**EEVDF** 代替了 2.6的 FCS。



## 5. System Calls

比较熟悉，先略过。



## 6. Kernel Data Structures

一些基本宏

```c
#define container_of(ptr, type, member)

// 获取type结构体的首地址

struct Node {
    int a;
    int b;
}
Node a = {1, 2};

Node* node = container_of(&a.b, struct Node, b); // 通过container_of，拿到所属结构体的首地址
```



### 6.1 Linked Lists

链表定义

```c
struct list_head {
	struct list_head *next, *prev;
};
```

使用方法如下，假设有个fox链表

```c
struct fox {
	unsigned long tail_length;  /* length in centimeters of tail */
	unsigned long weight;       /* weight in kilograms */
	bool 		  is_fantastic; /* is this fox fantastic? */
	struct list_head list;      /* list of all fox structures */  // 指向上/下一个fox
};
```

这样，通过```container_of```，可以拿到list节点真正的数据。

```c
#define list_entry(ptr, type, member) container_of(ptr, type, member)
```

初始化一个list节点

```c
static inline void INIT_LIST_HEAD(struct list_head *list)
{
	list->next = list;
	list->prev = list;
}
// 使用方法如下

void demo() {
    struct fox *red_fox;
    red_fox = kmalloc(sizeof(*red_fox), GFP_KERNEL);
    red_fox->tail_length = 40;
    red_fox->weight = 6;
    red_fox->is_fantastic = false;
    INIT_LIST_HEAD(&red_fox->list);  // next/prev指向自己
}
```

创建一个list的方式

```c
#define LIST_HEAD_INIT(name) { &(name), &(name) }

#define LIST_HEAD(name) \
	struct list_head name = LIST_HEAD_INIT(name)

// 使用方法
{
    LIST_HEAD(fox_list); // 创建了一个名为fox_list的链表，这个链表节点没有实际数据，类似dummy_head
}
```

向list添加节点

```c
list_add(struct list_head *new, struct list_head *head)

// 使用方法
{
    list_add(&f->list, &fox_list); // 将f直接加在fox_list后面
    // 相当于 从 fox_list <-> a <-> b 变成 fox_list <-> f <-> a <-> b
}    
```

删除节点

```c
list_del(struct list_head *entry)
    
// 使用方法
{
    list_del(&f->list); // 注意是直接传要删除的节点，不需要传fox_list
}
```

遍历list

```c
#define list_for_each(pos, head) \
	for (pos = (head)->next; pos != (head); pos = pos->next)

// 使用方法
{
    struct list_head *p;
	struct fox *f;
    list_for_each(p, &fox_list) {
		/* f points to the structure in which the list is embedded */
		f = list_entry(p, struct fox, list);
	}	
}
```

另一种更多使用的遍历方法，宏内已经帮助处理list_entry了

```c
#define list_first_entry(ptr, type, member) \
	list_entry((ptr)->next, type, member)
	
#define list_next_entry(pos, member) \
	list_entry((pos)->member.next, typeof(*(pos)), member)	

#define list_for_each_entry(pos, head, member)				\
	for (pos = list_first_entry(head, typeof(*pos), member);	\
	     &pos->member != (head);					\
	     pos = list_next_entry(pos, member))
	     
// 使用方法
{
    struct fox *f;
    list_for_each_entry(f, &fox_list, list) {
		/* on each iteration, ‘f’ points to the next fox structure ... */
	}
}
```



### 6.2 Queues

Linux的FIFO叫kfifo 

```c
struct __kfifo {
	unsigned int	in;
	unsigned int	out;
	unsigned int	mask;
	unsigned int	esize;
	void		*data;
};
```



创建kfifo

```c
int kfifo_alloc(struct kfifo *fifo, unsigned int size, gfp_t gfp_mask); // size需要是2的幂
```

入队

```c
// 从from入队len个字节到fifo，新版本Linux未必是以函数实现kfifo_in，但参数和功能不变
unsigned int kfifo_in(struct kfifo *fifo, const void *from, unsigned int len); 
```

出队

```c
// 复制len个字节到to，且出队
unsigned int kfifo_out(struct kfifo *fifo, void *to, unsigned int len);

// 只看数据，不出队
unsigned int kfifo_out_peek(struct kfifo *fifo, void *to, unsigned int len, unsigned offset);
```

可以看出，kfifo是以字节为单位管理的，而上一节的链表可以隐式帮助我们转换类型。

### 6.3 Maps

Linux中的map不是通常意义上的map，而是提供id到pointer的转换服务，结构体名称为idr。这里的id可以是file descriptors, process IDs, packet identifiers in networking protocols, SCSI tags and device instance numbers.

```c
struct idr {
	struct radix_tree_root	idr_rt;
	unsigned int		idr_base;
	unsigned int		idr_next;
};
```

IDR提供生成id的服务，且像map一样帮助存储id到pointer的映射关系。

目前IDR已被废弃，改用xArray (eXtensible Arrays)



## 7. Interrupts and Interrupt Handlers

中断分为上半部和下半部，上半部快速处理紧要的事情，下半部在后台处理耗时的事情，本章介绍上半部。



注册中断处理函数

```c
// irq就是中断号，handler是函数指针
int request_irq(unsigned int irq, irq_handler_t handler, unsigned long flags,
	    const char *name, void *dev)

// 函数指针类型    
typedef irqreturn_t (*irq_handler_t)(int, void *);

// 取消注册
void free_irq(unsigned int irq, void *dev)
```

一些重要的flag

* **IRQF_DISABLED**  When set, this flag instructs the kernel to disable all interrupts when executing this interrupt handler.When unset, interrupt handlers run with all interrupts except their own enabled.
* **IRQF_SAMPLE_RANDOM** This flag specifies that interrupts generated by this device should contribute to the kernel entropy pool. The kernel entropy pool provides truly random numbers derived from various random events.
* **IRQF_TIMER** This flag specifies that this handler processes interrupts for the system timer.
* **IRQF_SHARED** This flag specifies that the interrupt line(Physical or logical connection used to convey interrupt requests between devices and the CPU or interrupt controller.) can be shared among multiple interrupt handlers. Each handler registered on a given line must specify this flag; otherwise, only one handler can exist per line.(一种理解是多个设备共享中断号，这个时候dev参数不能为NULL且必须唯一，同时dev需要提供一个功能告知其有没有发起中断)

中断处理程序返回irqreturn，有下面几个值

* **IRQ_NONE** interrupt was not from this device or was not handled
* **IRQ_HANDLED** 中断已处理
* **IRQ_WAKE_THREAD** handler requests to wake the handler thread(应该是表示还有中断下半部？)

Linux一般不用处理嵌套中断，但是可以同时处理不同的中断。

设备是发中断给interrupt controller，而不是直接给processor。 中断的控制往往是和硬件结构相关的。



## 8. Bottom Halves and Deferring Work

**Softirqs**

Softirqs是在编译时注册的。

```c
// linux/interrupt.h
struct softirq_action
{
	void	(*action)(struct softirq_action *);
};

// kernel/softirq.c
static struct softirq_action softirq_vec[NR_SOFTIRQS]

const char * const softirq_to_name[NR_SOFTIRQS] = {
	"HI", "TIMER", "NET_TX", "NET_RX", "BLOCK", "IRQ_POLL",
	"TASKLET", "SCHED", "HRTIMER", "RCU"
};
```

softirq不抢占softirq，且相同类型的softirq可以在不同的核同时运行，所以需要注意保护共享数据。

在```__do_softirq()``` 函数里检查当前是否有softirq任务要做，通过```pending = local_softirq_pending();``` pending的某个bit为1，则说明对应位置的softirq要处理。目前只有网络或者块设备等少数模块会使用softirq。中断上半部通过```raise_softirq(NET_TX_SOFTIRQ);``` 发起softirq。

**Tasklets**



tasklets是在softirq之上建立的机制，一般比softirq更简单易用。softirq一般用于高频率且适合多线程的任务，比如收发网络包。

因为tasklets在softirq基础之上建立，所以有两个softirq的编号，HI_SOFTIRQ和TASKLET_SOFTIRQ，其中HI_SOFTIRQ代表优先级更高的，其他都一样。

Tasklet结构体如下

(tasklet_struct已经被标注为deprecated，推荐使用threaded IRQs，[区别](https://stackoverflow.com/questions/68616407/what-is-the-difference-between-threaded-interrupt-handler-and-tasklet))

```c
struct tasklet_struct
{
	struct tasklet_struct *next;
	unsigned long state;
	atomic_t count;  // 如果非0，tasklet关闭,不执行 
	bool use_callback;
	union {
		void (*func)(unsigned long data);
		void (*callback)(struct tasklet_struct *t);
	};
	unsigned long data;  // func的参数
};
```

发起的tasklets被存储在两个per-processor结构中: tasklet_vec  和 tasklet_hi_vec (for high-priority tasklets)。发起tasklet用```tasklet_schedule(struct tasklet_struct *t)``` 和 ```tasklet_hi_schedule(struct tasklet_struct *t)```。这两个函数内部将参数tasklet放入tasklet_vec，然后按softirq的方式发起TASKLET_SOFTIRQ，于是在softirq的处理函数中依次执行HI_SOFTIRQ中的函数。

As with softirqs, tasklets cannot sleep.This means you cannot use semaphores or other blocking functions in a tasklet.Tasklets also run with all interrupts enabled.

**Work Queues**

work queues are schedulable and can therefore sleep. 运行在内核线程。

## 9. An Introduction to Kernel Synchronization
主要介绍同步的概念

## 10. Kernel Synchronization Methods
### 10.1 Atomic Operations

```c
typedef struct {
	int counter;
} atomic_t;
```

内核还提供bit的原子操作

### 10.2 Spin Locks

spin lock可以用在interrupt handlers，但是使用前需要关闭当前core的中断，但是关之前需要记录当前的状态。

### 10.3 Semaphores

Semaphores in Linux are sleeping locks.When a task attempts to acquire a semaphore that is unavailable, the semaphore places the task onto a wait queue and puts the task to sleep.

必须用在process context，因为中断处理程序是不被调度的。

semaphores允许有多个holder

```c
struct semaphore {
	raw_spinlock_t		lock;
	unsigned int		count;
	struct list_head	wait_list;
};
```



### 10.4 Mutex

```c
struct mutex {
	atomic_long_t		owner;
	raw_spinlock_t		wait_lock;
};
```

### 10.5 其他



## 11. Timers and Time Management

### 11.1 Kernel Notion of Time

system timer每个tick触发一次时间中断，一个tick就是两次中断间的时间，是预设置的。tick rate是频率，因此tick = 1/(tick rate) 秒。有些关于时间的函数是需要每个tick都执行的。

### 11.2 The Tick Rate: HZ

x86默认timer是100HZ(作者那个2.6时代，在我的笔记本上是250)，也就是10ms一次tick(其实大多数处理器都是)

Linux曾在2.5将频率提高到1000HZ，但后来又改回来了。提高频率可以提高关于时间函数的精度，并且可能会提高性能，比如poll()和select()。

显然，更高的频率会有更多的关于时间中断的开销。但是在现代处理器上，1000HZ没有对性能的损耗不是unacceptable(可接受的~)

### 11.3 Jiffies

jiffies是一个全局变量(include/linux/jiffies.h    unsigned long)，存储系统启动到现在的tick数。64位的jiffies几乎不可能溢出。

为了保证user space读时间的准确性，另有一个USER_HZ变量，可以从HZ转到USER_HZ

### 11.4 Hardware Clocks and Timers

实际上有两个关于时间的硬件，一个是上面讨论的system timer，还有一个real-time Clock(RTC).  RTC即使在关机的时候也正常记录时间，于是开机时可以读RTC，这也是RTC的主要职责。

x86主要的system timer是programmable interrupt timer (PIT)

### 11.5 The Timer Interrupt Handler

主要分为两个部分，首先是和硬件相关(architecture-dependent)，然后与硬件无关

**architecture-dependent**

获取xtime_lock锁，可以安全更新Jiffies和xtime，确认并重置中断，可能保存时间到RTC等等

**architecture-independent**

增加jiffies计数1，为当前进程更新资源使用，运行过期的dynamic timers，执行scheduler_tick()，计算load average等。这些在```tick_periodic()```中

### 11.6 Timers

```c
struct timer_list {
	struct list_head entry;  // entry in linked list of timers 
	unsigned long expires;   // expiration value, in jiffies
	void (*function)(unsigned long); // the timer handler function
	unsigned long data;      // lone argument to the handler
	struct tvec_t_base_s *base; // internal timer field, do not touch
};
```

有趣的是，timer队列并不是有序的，也不是树形结构，而是"the timer wheels"，具体实现可阅读 https://lwn.net/Articles/156329/



## 12. Memory Management



## 13. The Virtual Filesystem

### 13.1 Unix Filesystems

Unix文件系统中，文件的metadata和文件的内容不在一起，而是存储在称为inode(index node)的地方。所有的文件metadata存在一起的地方叫superblock。superblock除了各个文件的metadata，还包含整个filesystem的metadata。

### 13.2 VFS Objects and Their Data Structures

The four primary object types of the VFS are

* The *superblock* object, which represents a specific mounted filesystem.
* The *inode* object, which represents a specific file.
* The *dentry* object, which represents a directory entry, which is a single component of a path.
* The *file* object, which represents an open file as associated with a process.

fs_struct描述文件系统，file描述文件

### 13.3 The Superblock Object

每个文件系统都要有一个 `struct super_block`

一些常见操作有

* `struct inode * alloc_inode(struct super_block *sb)`
* `void destroy_inode(struct inode *inode)`
* `void dirty_inode(struct inode *inode)`  Invoked by the VFS when an inode is dirtied
* `void write_inode(struct inode *inode, int wait)`
* `void delete_inode(struct inode *inode)`
* `int sync_fs(struct super_block *sb, int wait)`  Synchronizes filesystem metadata with the on-disk filesystem.
* `void clear_inode(struct inode *inode)` Called by the VFS to release the inode and clear any pages containing related data.

### 13.4 The Inode Object

`struct inode`

### 13.5 The Dentry Object

Dentry objects are all components in a path, including files. 

 `struct dentry` does not correspond to any sort of on-disk data structure.

dentry用于快速查找path，有dentry cache，否则会从disk读inode信息，然后再进行字符串比较，性能差。dentry cache的存在也就相当于inode cache的存在。

### 13.6 The File Object

一个文件可以同时有多个File结构体，但是File内部指向同一个dentry，也就指向同一个inode。File本身也是纯内存结构，没有保存在磁盘上。

## 14. The Block I/O Layer

block devices 指能随机访问固定大小的硬件， char devices则必须顺序访问，比如键盘。

*sector* 硬件可寻址的最小单位，通常512byte

*block* 操作系统定义的最小单位，不超过page，不小于sector

*page* 操作系统定义

每个*buffer* (在内存中)包含一个block，每个buffer有一个buffer head，包含一些管理信息。

一个page可以有多个buffer。

每次block I/O请求至少有一个bio结构体，其中包含多个block。

### 14.1 I/O Schedulers

磁盘操作是非常慢的，所以内核不会立即处理发起的IO请求。而是会 merging 和 sorting 请求来优化性能。

Linux同时有多种Schedulers可供选择，默认是CFQ，每个进程一个queue，里面保存排序后的IO请求，每次轮流从各个queue中取一些请求执行。



## 16. The Page Cache and Page Writeback
page cache是对disk的cache，在Linux中，对page修改后不会直接写入磁盘，而是采用write back的策略，一段时间后再写回。

Linux采用缓存过期策略采用*two-list strategy*，一个是active list，一个是inactive list，两个list各自采用类似LRU的策略，但是当active list过大时，会将其中部分页转到inactive list。只有在inactive list中的page才可以被替换。page要进入active的前提是，当被访问时，它已经在inactive list中。这样的算法可以避免那些只访问一次的页被保留。