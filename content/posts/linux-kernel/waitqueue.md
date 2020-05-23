---
title: "Waitqueue"
date: 2020-05-06T20:00:07+08:00
description: ""
draft: true
tags: []
categories: []
---

wait queue思想比较简单，但是涉及到的多核和锁的问题比较多，这些问题也很复杂，因此本文并不会涉及这些内容。本文主要梳理wait queue的基本思想，以及一些驱动程序如何使用wait queue，包括常见的epoll驱动和uio驱动。

### 简介

wait queue采用双向链表的方式管理等待的进程，当事件发生的时候，会调用`wake up`函数，该函数会从等待队列中唤醒进程。`wait queue`主要提供了以下几个结构体和方法：

`wait queue`主要提供了两个结构体：

```c
// 队列元素
struct wait_queue_entry {
	unsigned int		flags;
	void			*private;
	wait_queue_func_t	func;	//回调函数，用于唤醒进程
	struct list_head	entry;
};

// 队首，一个队首往往代表着一个事件，该队首代表的队列中的队列元素就是等待该事件发生的进程
struct wait_queue_head {
	spinlock_t		lock;
	struct list_head	head;
};
```

`wait queue`主要提供了八个基本方法：

```c
// 创建一个新的wait queue entry，包含声明和初始化
#define DECLARE_WAITQUEUE(name, tsk)
// 创建一个新的wait queue，包含声明和初始化
#define DECLARE_WAIT_QUEUE_HEAD(name)
// 初始化wait queue
#define init_waitqueue_head(wq_head)
// 初始化wait queue entry
static inline void init_waitqueue_entry(struct wait_queue_entry *wq_entry, struct task_struct *p)

// 向wait queue添加wait queue entry
void add_wait_queue(struct wait_queue_head *wq_head, struct wait_queue_entry *wq_entry)
// 向wait queue添加wait queue entry，该entry带有WQ_FLAG_EXCLUSIVE特性，防止惊群问题，导致性能下降
void add_wait_queue_exclusive(struct wait_queue_head *wq_head, struct wait_queue_entry *wq_entry)
// 从wait queue移除一个entry，一般是唤醒之后的操作
void remove_wait_queue(struct wait_queue_head *wq_head, struct wait_queue_entry *wq_entry)
// 遍历wait queue，唤醒nr_exclusive个进程：回调函数default_wake_function，调用关系比较复杂，主要是将该进程挂载到run_queue以及将其状态置为TASK_RUNNING
static int __wake_up_common(struct wait_queue_head *wq_head, unsigned int mode, int nr_exclusive, int wake_flags, void *key, wait_queue_entry_t *bookmark)
```

### 历史

wait queue早期的策略比较简单。当事件发生时，调用`wake_up`函数唤醒所有的在该事件的阻塞进程。但是会有一个严重的问题，就是惊群问题。惊群问题是指多个被唤醒的进程竞争**同一个临界资源**。wait queue添加`exclusive wait`特性，使得每次只有一个进程被唤醒（如果不是临界资源可以不使用`exclusive wait`特性）。随着逐渐发展，wait queue变得越来越复杂，尤其是其回调函数


Waiting / Blocking in Linux Driver Part – 3 https://sysplay.in/blog/linux-kernel-internals/2015/12/waiting-blocking-in-linux-driver-part-3/



### epoll wait queue的使用



### uio wait queue的使用

uio使用wait queue机制完成向用户驱动通知某个中断号的中断事件，当用户驱动程序读`/proc/irq/irq num/irq`文件时，如果中断没有发生时会被阻塞。当中断发生时，中断处理函数会唤醒该进程：

```
global wqh
global condition
func init
	register_irq(irq,irq_handler)
	wqh = init_wait_queue_head
	condition = false

func irq_handler
	condition = true
	wake_up(wqh)

func read:
	add_wait_queue(current, wqh)
	while
		set_current_state(TASK_INTERRUPTIBLE)
		if condition is true
			set_current_state(TASK_RUNNING)
			break
		schedule
	set_current_state(TASK_RUNNING)
	remove_wait_queue(current)
```

当调用`int __uio_register_device(struct module *owner, struct device *parent, struct uio_info *info)`函数注册一个uio驱动时，初始化该wait queue，并注册中断处理函数，下面是代码片段：

```c
int __uio_register_device(struct module *owner,
			  struct device *parent,
			  struct uio_info *info)
	struct uio_device *idev;
	idev = kzalloc(sizeof(*idev), GFP_KERNEL);
	// 初始化一个wait queue head
	init_waitqueue_head(&idev->wait);
	// wait queue head和中断事件绑定
	ret = request_irq(info->irq, uio_interrupt,
				  info->irq_flags, info->name, idev);
```

用户驱动需要自己开一个单独的线程去接收中断事件，避免整个进程被阻塞。该线程通过读`/proc/irq/irq number/irq`文件，下面时读时执行的代码。流程如下：

1. 首先通过`add_wait_queue`将自己挂载到wait queue上；
2. `set_current_state`将自己的状态设置为TASK_INTERRUPTIBLE，避免自己在事件没到来之前就被唤醒，导致无用的上下文切换；
3. 当前仍然处于该进程的上下文，通过`event_count`是否增加来判断中断是否发生，如果发生了则重新将进程挂载到run queue，否则发起调度，等待通知，即`wake_up`；

```c
static ssize_t uio_read(struct file *filep, char __user *buf,
			size_t count, loff_t *ppos)
{
	struct uio_listener *listener = filep->private_data;
	struct uio_device *idev = listener->dev;
	DECLARE_WAITQUEUE(wait, current);
	ssize_t retval = 0;
	s32 event_count;

	add_wait_queue(&idev->wait, &wait);

	do {
		// 将当前进程状态设置为TASK_INTERRUPTIBLE，可被信号唤醒
		set_current_state(TASK_INTERRUPTIBLE);
		// 读取当前事件次数
		event_count = atomic_read(&idev->event);
		if (event_count != listener->event_count) {
			// 说明发生了新的事件，可以通知进程去处理
			__set_current_state(TASK_RUNNING);
			if (copy_to_user(buf, &event_count, count))
				retval = -EFAULT;
			else {
				listener->event_count = event_count;
				retval = count;
			}
			break;
		}
		// 如果是非阻塞读，则直接返回
		if (filep->f_flags & O_NONBLOCK) {
			retval = -EAGAIN;
			break;
		}
		// 被信号唤醒
		if (signal_pending(current)) {
			retval = -ERESTARTSYS;
			break;
		}
		// 调度，等待irq_handler将自己唤醒
		schedule();
	} while (1);
	// 发生新的事件，将进程挂载到run queue
	__set_current_state(TASK_RUNNING);
	// 将该进程从wait queue移除
	remove_wait_queue(&idev->wait, &wait);

	return retval;
}
```

因为是捕捉中断事件，所以需要像该中断号注册中断处理函数，然后在中断处理函数中唤醒wait queue上的等待进程。中断处理函数注册是在`__uio_register_device`函数完成的，下面我们关注中断发生时中断处理函数的动作：

```c
static irqreturn_t uio_interrupt(int irq, void *dev_id)
{
	struct uio_device *idev = (struct uio_device *)dev_id;
	// 增加中断发生的次数，因为在wake_up后，进程是通过event次数来判断是否发生中断
	atomic_inc(&idev->event);
	// 唤醒该进程
	wake_up_interruptible(&idev->wait);
}
```

### 使用wait queue提供的wait_events


```c
#include <linux/module.h>
#include <linux/kernel.h>
#include <linux/fs.h>
#include <linux/cdev.h>
#include <linux/device.h>
#include <linux/errno.h>
#include <asm/uaccess.h>
#include <linux/wait.h>
#include <linux/sched.h>
#include <linux/delay.h>

#define FIRST_MINOR 0
#define MINOR_CNT 1

static char flag = 'n';
static dev_t dev;
static struct cdev c_dev;
static struct class *cl;
static DECLARE_WAIT_QUEUE_HEAD(wq);

int open(struct inode *inode, struct file *filp)
{
	printk(KERN_INFO "Inside open\n");
	return 0;
}

int release(struct inode *inode, struct file *filp) 
{
	printk (KERN_INFO "Inside close\n");
	return 0;
}

ssize_t read(struct file *filp, char *buff, size_t count, loff_t *offp) 
{
	printk(KERN_INFO "Inside read\n");
	printk(KERN_INFO "Scheduling Out\n");
	wait_event_interruptible(wq, flag == 'y');
	flag = 'n';
	printk(KERN_INFO "Woken Up\n");
	return 0;
}

ssize_t write(struct file *filp, const char *buff, size_t count, loff_t *offp) 
{   
	printk(KERN_INFO "Inside write\n");
	if (copy_from_user(&flag, buff, 1))
	{
		return -EFAULT;
	}
	printk(KERN_INFO "%c", flag);
	wake_up_interruptible(&wq);
	return count;
}

struct file_operations pra_fops = {
	read:        read,
	write:       write,
	open:        open,
	release:     release
};

int wq_init (void)
{
	int ret;
	struct device *dev_ret;

	if ((ret = alloc_chrdev_region(&dev, FIRST_MINOR, MINOR_CNT, "SCD")) < 0)
	{
		return ret;
	}
	printk("Major Nr: %d\n", MAJOR(dev));

	cdev_init(&c_dev, &pra_fops);

	if ((ret = cdev_add(&c_dev, dev, MINOR_CNT)) < 0)
	{
		unregister_chrdev_region(dev, MINOR_CNT);
		return ret;
	}

	if (IS_ERR(cl = class_create(THIS_MODULE, "chardrv")))
	{
		cdev_del(&c_dev);
		unregister_chrdev_region(dev, MINOR_CNT);
		return PTR_ERR(cl);
	}
	if (IS_ERR(dev_ret = device_create(cl, NULL, dev, NULL, "mychar%d", 0)))
	{
		class_destroy(cl);
		cdev_del(&c_dev);
		unregister_chrdev_region(dev, MINOR_CNT);
		return PTR_ERR(dev_ret);
	}
	return 0;
}

void wq_cleanup(void)
{
	printk(KERN_INFO "Inside cleanup_module\n");
	device_destroy(cl, dev);
	class_destroy(cl);
	cdev_del(&c_dev);
	unregister_chrdev_region(dev, MINOR_CNT);
}

module_init(wq_init);
module_exit(wq_cleanup);

MODULE_LICENSE("GPL");
MODULE_AUTHOR("Pradeep");
MODULE_DESCRIPTION("Waiting Process Demo");
```

