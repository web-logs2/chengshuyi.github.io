---
title: "Poll_select_epoll"
date: 2020-05-05T13:06:02+08:00
description: ""
draft: true
tags: []
categories: []
---


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
```

poll系统调用：
    将毫秒转换成内核struct timespec64时间结构体行态
    call do_sys_poll
        将用户传进来的struct pollfd数组拷贝到内核空间，用struct poll_list管理，每个poll_list占用4K的内存
        1. 初始化struct poll_wqueues，主要是记录当前的polling_task和poll_table的_qproc
        call do_poll
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
```



```
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
        schedule_hrtimeout_range





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

       · the timeout expires.
