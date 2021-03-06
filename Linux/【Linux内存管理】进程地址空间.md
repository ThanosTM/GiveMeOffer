## 进程地址空间的管理
#### 与内核内存管理的不同
内核以直接了当的方式获得动态内存：`alloc_pages()`从分区页框分配器中获得页框，`kmem_cache_alloc()`使用slab分配器为专用或通用对象分配块，`vmalloc()`获得一块非连续的内存区。当请求的内存区得以满足时，均返回一个页描述符地址或线性地址。内核分配内存的简单之处是基于一下两个原因的：
- 内核时OS中优先级最高的，没有理由试图推迟这个请求。
- 内核充分信任自己，不必插入针对编程错误的保护措施。

而给用户态进程分配内存时：
- 内核尽量推迟给用户态进程分配内存
- 不可信任的，要提供严格审查并随时准备捕获用户态进程的寻址错误。

#### 添加或删除线性区以动态地修改进程的地址空间
进程获得新线性区的典型情况：
- shell创建新进程去执行某命令
- 载入完全不同的程序（exec）
- 对某个文件或某个部分作内存映射
- 向堆/栈增加数据
- 创建IPC共享线性区与其他进程共享合作

## 内存描述符
#### `mm_struct`
```c
struct mm_struct {
	struct vm_area_struct * mmap;		/* list of VMAs */
	struct rb_root mm_rb;
	struct vm_area_struct * mmap_cache;	/* last find_vma result */
#ifdef CONFIG_MMU
	unsigned long (*get_unmapped_area) (struct file *filp,
				unsigned long addr, unsigned long len,
				unsigned long pgoff, unsigned long flags);
	void (*unmap_area) (struct mm_struct *mm, unsigned long addr);
#endif
	unsigned long mmap_base;		/* base of mmap area */
	unsigned long task_size;		/* size of task vm space */
	unsigned long cached_hole_size; 	/* if non-zero, the largest hole below free_area_cache */
	unsigned long free_area_cache;		/* first hole of size cached_hole_size or larger */
	pgd_t * pgd;
	atomic_t mm_users;			/* How many users with user space? */
	atomic_t mm_count;			/* How many references to "struct mm_struct" (users count as 1) */
	int map_count;				/* number of VMAs */
	struct rw_semaphore mmap_sem;
	spinlock_t page_table_lock;		/* Protects page tables and some counters */

	struct list_head mmlist;		/* List of maybe swapped mm's.	These are globally strung
						 * together off init_mm.mmlist, and are protected
						 * by mmlist_lock
						 */


	unsigned long hiwater_rss;	/* High-watermark of RSS usage */
	unsigned long hiwater_vm;	/* High-water virtual memory usage */

	unsigned long total_vm, locked_vm, shared_vm, exec_vm;
	unsigned long stack_vm, reserved_vm, def_flags, nr_ptes;
	unsigned long start_code, end_code, start_data, end_data;
	unsigned long start_brk, brk, start_stack;
	unsigned long arg_start, arg_end, env_start, env_end;

	unsigned long saved_auxv[AT_VECTOR_SIZE]; /* for /proc/PID/auxv */

	/*
	 * Special counters, in some configurations protected by the
	 * page_table_lock, in other configurations by being atomic.
	 */
	struct mm_rss_stat rss_stat;

	struct linux_binfmt *binfmt;

	cpumask_t cpu_vm_mask;

	/* Architecture-specific MM context */
	mm_context_t context;

	/* Swap token stuff */
	/*
	 * Last value of global fault stamp as seen by this process.
	 * In other words, this value gives an indication of how long
	 * it has been since this task got the token.
	 * Look at mm/thrash.c
	 */
	unsigned int faultstamp;
	unsigned int token_priority;
	unsigned int last_interval;

	unsigned long flags; /* Must use atomic bitops to access the bits */
};
```

与进程地址空间有关的全部信息包含在`mm_struct`中，所有内存描述符存放在一个双向链表中；以下是一些较为重要的字段：
- `mm_users`：存放共享该`mm_struct`数据结构的轻量级进程即线程的个数；
- `mm_count`：内存描述符的主使用计数器，在`mm_users`次使用计数器的所有用户在`mm_count`中只作为一个单位；当`mm_count`递减为0时，都要解除这个内存描述符。

下面说明`mm_users`和`mm_count`的区别：假设有一个用户态进程A，由两个轻量级进程所共享，则`mm_users`=2，`mm_count`=1；当内存描述符暂借给内核线程后，`mm_count`递增为2，这时即使两个轻量级线程都死亡，这个内存描述符也不会释放，直到内核线程使用完毕。

#### 内核线程的内存描述符
由于内核不使用线性区，因此内存描述符的很多字段对于内核线程时没有意义的；为了避免无用的TLB和硬件cache的刷新，内核线程使用一组最近运行的普通进程的页表，于是在每个进程描述符中包含了两种内存描述符指针：mm和active_mm。对于普通线程，mm和active_mm相同，对于内核线程，mm为NULL，active_mm为上一个运行的进程的active_mm.

## 线性区
#### 线性区数据结构
Linux通过`struct vm_area_struct`实现线性区，较为重要的字段有：
- `vm_start`、`vm_end`：区间的第一个线性地址、区间之外的第一个线性地址。差表示线性区的长度；
- `vm_mm`：指向拥有这个区间的进程的内存描述符`mm_struct`；
- `vm_next`：线性区链表的下一个线性区
- `vm_flags`：线性区标志，包含了线性区的权限等，多个标志可以任意组合
- `vm_rb`：用于红黑树组织的数据

#### 组织图
进程地址空间、内存描述符、线性区链表三者之间的关系图：
![image](https://www.linuxidc.com/upload/2011_04/110416063664514.gif)

#### 红黑树管理内存描述符
- 原因：尽管多数Linux进程使用的线性区非常少，但诸如面向对象的数据库、malloc专用调试器等庞大的应用程序会有成百上千个线性区，此时线性区链表的管理显得非常低效。
- Linux既使用了链表又使用了红黑树，红黑树用于快速确定含有指定地址的线性区，内核通过该结点的前后元素可以快速更新链表而不用扫描链表；当需要扫描整个线性区几个时，这是通常使用链表进行操作。

#### 分配与释放线性地址区间
内核确保进程所拥有的线性区不会重叠，并且尽力将新分配的线性区与紧邻的现有线性区进行合并。如果两个相邻区访问权限相匹配，则进行合并。当某些情况下需要删除某个线性地址区间时，可能会将某个线性区分成两个更小的部分。

###### 分配`do_mmap()`
流程：
- 检查参数值是否正确，请求是否能够满足
- `get_unmapped_area()`获得新线性区的地址区间
- `find_vma_prepare()`获得处于新区间之前的线性区对象的位置，以及在红黑树中新线性区的位置。
- 尝试是否可以以现有的线性区扩展来包含新的区间，这要求vm_flags标志相同，若能够成功扩展，也就不需要进行下面的新线性区数据结构的分配了。
- 调用`kmem_cache_alloc()`分配一个`vm_area_struct`数据结构，并为这个新线性区初始化
- 插入线性区链表和红黑树，增加进程地址空间大小。
- 返回新线性区地址

###### 释放`do_munmap()`
复杂之处在于：要删除的区间并不总对应与一个线性区，有可能是一个线性区的某一部分，或者跨越多个线性区。

## 缺页异常处理程序
简化版的流程图，实际上比这个更为复杂：
![image](https://pic2.zhimg.com/v2-6d8dad5dc9640d60b2f36a0d00260ac1_r.jpg)

#### error_code
缺页异常处理主要通过这3位异常码决定执行的动作：
- 第0位：0-访问一个不存在的页，1-页存在但访问权限无效
- 第1位：0-读，1-写
- 第2位：0-内核态，1-用户态

#### 处理地址空间以外的错误地址`bad_area`
- 用户态：发送SIGSEGV段错误
- 内核态：由于把某个线性地址作为系统调用的参数传递给内核，发送SIGSEGV至修正代码；或者由于真正的内核缺陷，杀死当前进程。

#### 处理地址空间内的错误地址`good_area`
前面一系列检查省略，主要是调用`handle_mm_fault()`分配一个新的页框，若被访问的页不存在，则将执行==请求调页==，若存在但是标记为只读，则执行==写时复制==。

#### 请求调页
Linux内核将分配页框推迟到进程访问的页不再内存中为止，背后的动机是进程开始运行的时候并不访问其地址空间的全部地址，由于程序的局部性，真正使用的页框只有一小部分，其余地址空间可以由其他进程来使用。代价是系统额外的开销，因为缺页异常需要进入到内核态，但局部性原理确保了==缺页是一种稀有事件==，因此请求调页总体上是系统具有更高的吞吐量。

#### 写时复制
第一代Unix采用的`fork()`将父进程的整个地址空间复制给子进程，而这种非常耗时；现代的Linux采用写时复制技术，父子进程共享页框，同时将共享的页框置位只读，当任何一个进程试图写该页框时产生异常，此时产生分离，分配新页框并把原页框内的内容进行拷贝。

## 创建与删除进程的地址空间
创建一个进程时调用`copy_mm()`函数，通过建立新进程的所有页表和内存描述符来创建进程的地址空间。

- 传统进程：继承父进程的地址空间，当其中一个进程试图对某个共享页进行写时，则执行写时复制。
- 轻量级进程：使用父进程的地址空间，需要谨慎协调父子进程对同一内存的访问。

