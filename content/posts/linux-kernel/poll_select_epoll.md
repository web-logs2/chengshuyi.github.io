---
title: "poll select epoll对比分析"
date: 2020-05-05T13:06:02+08:00
description: ""
draft: false
tags: [linux内核]
categories: [linux内核]
---

文章[^3]主要提到epoll需要新增几个syscall，来满足某些应用的需求（如qemu）。

* `epoll_ctl_batch`：支持更新批量的fd，之前的`epoll_ctl`一次只能更新一个fd；
* `epoll_pwait1`：解决`epoll_pwait`的时间精度不高（毫秒级别）；
* 添加`EPOLLEXCLUSIVE`标识，解决惊群效应，对应的wait queue采用`add_wait_queue_exclusive`；
* 但是，这样会导致wait queue一直唤醒等在队首的进程，因此增加了`add_wait_queue_rr`功能（**没怎么理解，既然在等待队列上，说明该进程是空闲的，空闲的去执行任务不很正常吗？**）；

文章[^4]主要提到两个问题，一个是：惊群效应；另外一个是锁。

文章[^5]主要提到在用户空间和内核空间维护一个环形缓冲（mmap的方式）。因为目前监听的fd的事件主要通过`epoll_ctl`的方式来进行修改，每次修改的话都需要进行一次系统用，比较耗时。因此，提出了采用环形缓冲的方式来通知事件。相当于，不采用系统调用的方式同内核通信，而是采用共享内存的方式，来提高程序的效率。这种ring-buffer的方式比较常见，比如io-uring等等。所以，作者提到内核是否能够提供一个统一的ring-buffer的交互形式，可以让编程人员更方便的使用。注：貌似该patch最后并没有整合到内核版本。

下面首先对select poll epoll的系统调用、支持的事件进行了对比，其次介绍了linux io复用的原理，最后从源码执行流程分析为什么epoll在监听大量描述符性能要好。

### 系统调用

poll核心的系统调用只有一个，其中参数`struct pollfd *fds`是一个链表，包含了要监听的所有文件，可以看到当监听的文件数量过多时，会发生大量的数据从用户空间拷贝到内核空间，但是**poll监听的描述符个数没有限制**。

`int poll(struct pollfd *fds, nfds_t nfds, int timeout);`

select同poll一样将整个fd的传给内核，只是select采用位图的方式。

```
int select(int nfds, fd_set *readfds, fd_set *writefds,
                  fd_set *exceptfds, struct timeval *timeout);

void FD_CLR(int fd, fd_set *set);		//将fd加入set集合
int  FD_ISSET(int fd, fd_set *set);		//将fd从set集合中清除
void FD_SET(int fd, fd_set *set);		//检测fd是否在set集合中，不在则返回0
void FD_ZERO(fd_set *set);				//将set清零使集合中不含任何fd
```

epoll主要提供了三个系统调用，其中`epoll_ctl`用来控制监听的fd列表，以及修改相关参数。

```c
int epoll_create(int size);
int epoll_ctl(int efd, int op, int fd, struct epoll_event *event);
int epoll_wait(int epfd, struct epoll_event *events, int maxevents, int timeout);
```

### 支持的事件

poll:

```
POLLIN： 表示对应的文件描述符可以读（包括对端SOCKET正常关闭）；
POLLPRI： 表示对应的文件描述符有紧急的数据可读（这里应该表示有带外数据到来）；
POLLOUT： 表示对应的文件描述符可以写；
POLLRDHUP：TCP连接被对方关闭；
POLLERR： 表示对应的文件描述符发生错误；
POLLHUP： 表示对应的文件描述符被挂断；
POLLNVAL：文件描述符未打开；
```

select:
```
POLLIN_SET：读
POLLOUT_SET：写
POLLEX_SET：异常
```

epoll:

```c
EPOLLIN ： 表示对应的文件描述符可以读（包括对端SOCKET正常关闭）；
EPOLLOUT： 表示对应的文件描述符可以写；
EPOLLPRI： 表示对应的文件描述符有紧急的数据可读（这里应该表示有带外数据到来）；
EPOLLERR： 表示对应的文件描述符发生错误；
EPOLLHUP： 表示对应的文件描述符被挂断；
EPOLLET： 将 EPOLL设为边缘触发(Edge Triggered)模式（默认为水平触发），这是相对于水平触发(Level Triggered)来说的。
EPOLLONESHOT： 只监听一次事件，当监听完这次事件之后，如果还需要继续监听这个socket的话，需要再次把这个socket加入到EPOLL队列里
```

### linux io复用基本原理

1. 首先需要理解的是：监听的描述符其实是将自身进程加入到该事件提供的等待队列上，事件到来时会从等待等列唤醒进程。比如：socket描述符包含了socket的描述信息，提供了poll方法，不论是select、poll或者epoll，调用socket提供的poll方法，然后由该poll方法将进程加入到socket等待队列上；
2. 其次是io复用，也就是利用select、poll或者epoll同时监听多个事件，事件到来时用户程序可以读取或者发送数据；
3. 最后要说的是，select、poll或者epoll各自有各自的优势。

### 源码分析

首先是select，下面是select脑图。其中比较关键的有两个点：

1. select有三类事件输入、输出和异常，用bitmap进行存储；
2. poll_initwait函数，特别注意`struct poll_wqueues`、`poll_table`、`poll_table_page`、`poll_table_entry`、`poll_queue_proc`几者之间的关系，以及作用；

```c
struct 	poll_wqueues {
	poll_table pt;							// 主要是一个回调函数，在调用vfs_poll时，具体会调用poll方法，poll方法回调该函数
	struct poll_table_page *table;			// 当内核栈不够用时，采用动态分配的poll_table_entry
	struct task_struct *polling_task;		// 当前的进程实体
	int triggered;
	int error;
	int inline_index;
	struct poll_table_entry inline_entries[N_INLINE_POLL_ENTRIES];	// 内核栈空间的poll_table_entry
};
```

3. 唤醒函数和poll一样，都是`pollwake`。

![](https://gitee.com/chengshuyi/scripts/raw/master/img/20200816111232.png)

接着是epoll，下面是epoll脑图。其中比较关键的有两个点：

1. epoll采用红黑树管理fd，方便插入和删除；
2. epoll同selet的`poll table`的回调函数不一样，epoll是ep_ptable_queue_proc，这样可以修该wait queue的回调函数，以及传入epoll entry；

![](https://gitee.com/chengshuyi/scripts/raw/master/img/20200816120550.png)

<!-- 
```c
struct pollfd {
	int fd;
	short events;
	short revents;
};

struct poll_list {
	struct poll_list *next;
	int len;
	struct pollfd entries[0];
};
// poll 可以监听不同的事件类型
typedef struct poll_table_struct {
	poll_queue_proc _qproc;
	__poll_t _key;
} poll_table;

struct poll_table_entry {
	struct file *filp;
	__poll_t key;
	wait_queue_entry_t wait;
	wait_queue_head_t *wait_address;
};

struct poll_wqueues {
	poll_table pt;
	struct poll_table_page *table;
	struct task_struct *polling_task;
	int triggered;
	int error;
	int inline_index;
    // 静态分配的poll_table_entry
	struct poll_table_entry inline_entries[N_INLINE_POLL_ENTRIES];
};

struct poll_table_page {
	struct poll_table_page * next;
	struct poll_table_entry * entry;
	struct poll_table_entry entries[0];
};
```

```c
static inline void poll_wait(struct file * filp, wait_queue_head_t * wait_address, poll_table *p)
``` -->

<!-- poll系统调用：
    将毫秒转换成内核struct timespec64时间结构体行态
    call do_sys_poll
        将用户传进来的struct pollfd数组拷贝到内核空间，用struct poll_list管理，每个poll_list占用4K的内存

初始化struct poll_wqueues，主要是记录当前的polling_task和poll_table的_qproc
 do_poll
遍历struct poll_list
    call do_pollfd
        call vfs_poll
            call file->f_op->poll(file, pt)
                调用file_operations的poll函数
                call socket->os->poll
                tcp_poll
                    call sock_poll_wait
                        call poll_wait
                            如果poll_table->_qproc为NULL直接返回，因为已经挂载过一次
                            call poll_table->_qproc，该值在编号1处赋值
                            __pollwait:将wait queue挂载到目标wait queue
                                通过poll_table获取struct poll_wqueues
                                从struct poll_wqueues获取struct poll_table_entry
                                初始化struct poll_table_entry，特别需要注意其挂载在sock的waitqueue上和其wait queue entry的回调函数是pollwake
                                初始化wait queue entry
                                call add_wait_queue
                                    将当前任务挂载到sock wait queue
                    将当前socket的状态返回
                返回socket的状态
            返回socket的状态
        返回值不为0，说明有事件发生；返回值为0，说明没有事件发生
        将poll_table->_qproc置为NULL，这样下次不会重复将其挂载到wait queue和重复的poll table entry
        call poll_schedule_timeout
call poll_schedule_timeout
    发起调度，timeout唤醒

当事件发生时，比如sock会修改其对应的状态，然后调用类似`wake_up`函数：

```
pollwake
    call __pollwake
        struct poll_wqueues triggered置为1，表示有事件发生
        call default_wake_function
            call try_to_wake_up
``` -->



<!-- ```
epoll_create
epoll_ctl

struct eventpoll {
	/*
	 * This mutex is used to ensure that files are not removed
	 * while epoll is using them. This is held during the event
	 * collection loop, the file cleanup path, the epoll file exit
	 * code and the ctl operations.
	 */
	struct mutex mtx;
	/* Wait queue used by sys_epoll_wait() */
	wait_queue_head_t wq;
	/* Wait queue used by file->poll() */
	wait_queue_head_t poll_wait;
	/* List of ready file descriptors */
	struct list_head rdllist;
	/* Lock which protects rdllist and ovflist */
	rwlock_t lock;
	/* RB tree root used to store monitored fd structs */
	struct rb_root_cached rbr;
	/*
	 * This is a single linked list that chains all the "struct epitem" that
	 * happened while transferring ready events to userspace w/out
	 * holding ->lock.
	 */
	struct epitem *ovflist;
	/* wakeup_source used when ep_scan_ready_list is running */
	struct wakeup_source *ws;
	/* The user that created the eventpoll descriptor */
	struct user_struct *user;
	struct file *file;
	/* used to optimize loop detection check */
	int visited;
	struct list_head visited_list_link;
};

struct epitem {
	union {
		/* RB tree node links this structure to the eventpoll RB tree */
		struct rb_node rbn;
		/* Used to free the struct epitem */
		struct rcu_head rcu;
	};

	/* List header used to link this structure to the eventpoll ready list */
	struct list_head rdllink;

	/*
	 * Works together "struct eventpoll"->ovflist in keeping the
	 * single linked chain of items.
	 */
	struct epitem *next;

	/* The file descriptor information this item refers to */
	struct epoll_filefd ffd;

	/* Number of active wait queue attached to poll operations */
	int nwait;

	/* List containing poll wait queues */
	struct list_head pwqlist;

	/* The "container" of this item */
	struct eventpoll *ep;

	/* List header used to link this item to the "struct file" items list */
	struct list_head fllink;

	/* wakeup_source used when EPOLLWAKEUP is set */
	struct wakeup_source __rcu *ws;

	/* The structure that describe the interested events and the source fd */
	struct epoll_event event;
};
struct epoll_filefd {
	struct file *file;
	int fd;
} __packed;
```

do_epoll_wait
    获取epoll file
    call ep_poll
        call ep_events_available
            检查struct eventpoll中rdllist是否存在元素
        call init_waitqueue_entry
        call __add_wait_queue_exclusive
        set_current_state(TASK_INTERRUPTIBLE);
        ep_events_available
        schedule_hrtimeout_range -->




<!-- 
epoll源码解析：https://blog.csdn.net/qq_36347375/article/details/91177145
int poll(struct pollfd *fds, nfds_t nfds, int timeout);
int ppoll(struct pollfd *fds, nfds_t nfds, const struct timespec *tmo_p, const sigset_t *sigmask);

nfds: The caller should specify the number of items in the fds array in nfds.
```c
struct pollfd {
    int   fd;         /* file descriptor */
    short events;     /* requested events */
    short revents;    /* returned events */
};
```

 POLLHUP, POLLERR, and POLLNVAL 

        The timeout argument specifies the number of milliseconds that poll()
       should block waiting for a file descriptor to become ready.  The call
       will block until either:
    
       · a file descriptor becomes ready;
    
       · the call is interrupted by a signal handler; or
    
       · the timeout expires. -->

<!-- ### 唤醒

从下面代码可以看到，select和poll均采用default_wake_function。`init_waitqueue_func_entry(&entry->wait, pollwake);`

```c
static int __pollwake(wait_queue_entry_t *wait, unsigned mode, int sync, void *key)
{
	struct poll_wqueues *pwq = wait->private;
	DECLARE_WAITQUEUE(dummy_wait, pwq->polling_task);

	/*
	 * Although this function is called under waitqueue lock, LOCK
	 * doesn't imply write barrier and the users expect write
	 * barrier semantics on wakeup functions.  The following
	 * smp_wmb() is equivalent to smp_wmb() in try_to_wake_up()
	 * and is paired with smp_store_mb() in poll_schedule_timeout.
	 */
	smp_wmb();
	pwq->triggered = 1;

	/*
	 * Perform the default wake up operation using a dummy
	 * waitqueue.
	 *
	 * TODO: This is hacky but there currently is no interface to
	 * pass in @sync.  @sync is scheduled to be removed and once
	 * that happens, wake_up_process() can be used directly.
	 */
	return default_wake_function(&dummy_wait, mode, sync, key);
}

static int pollwake(wait_queue_entry_t *wait, unsigned mode, int sync, void *key)
{
	struct poll_table_entry *entry;

	entry = container_of(wait, struct poll_table_entry, wait);
	if (key && !(key_to_poll(key) & entry->key))
		return 0;
	return __pollwake(wait, mode, sync, key);
}
/* Add a new entry */
static void __pollwait(struct file *filp, wait_queue_head_t *wait_address,
				poll_table *p)
{
	struct poll_wqueues *pwq = container_of(p, struct poll_wqueues, pt);
	struct poll_table_entry *entry = poll_get_entry(pwq);
	if (!entry)
		return;
	entry->filp = get_file(filp);
	// sock的waitqueue head
	entry->wait_address = wait_address;
	entry->key = p->_key;
	init_waitqueue_func_entry(&entry->wait, pollwake);
	entry->wait.private = pwq;
	add_wait_queue(wait_address, &entry->wait);
}
```

`init_poll_funcptr(&epq.pt, ep_ptable_queue_proc);`

`init_waitqueue_func_entry(&pwq->wait, ep_poll_callback);`

```c
init_poll_funcptr(&epq.pt, ep_ptable_queue_proc);

static void ep_ptable_queue_proc(struct file *file, wait_queue_head_t *whead,
				 poll_table *pt)
{
	struct epitem *epi = ep_item_from_epqueue(pt);
	struct eppoll_entry *pwq;

	if (epi->nwait >= 0 && (pwq = kmem_cache_alloc(pwq_cache, GFP_KERNEL))) {
		init_waitqueue_func_entry(&pwq->wait, ep_poll_callback);
		pwq->whead = whead;
		pwq->base = epi;
		if (epi->event.events & EPOLLEXCLUSIVE)
			add_wait_queue_exclusive(whead, &pwq->wait);
		else
			add_wait_queue(whead, &pwq->wait);
		list_add_tail(&pwq->llink, &epi->pwqlist);
		epi->nwait++;
	} else {
		/* We have to signal that an error occurred */
		epi->nwait = -1;
	}
}


``` -->



### 参考文献

[^3]: Epoll evolving. https://lwn.net/Articles/633422/.
[^4]: Issues with epoll(). https://lwn.net/Articles/637435/.
[^5]: A ring buffer for epoll. https://lwn.net/Articles/789603/.

