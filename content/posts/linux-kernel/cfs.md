---
title: "linux完全公平调度源码解析"
date: 2020-08-12T07:59:25+08:00
description: ""
draft: false
tags: [linux内核]
categories: [linux内核]
---

### cfs相关结构体

同cfs相关的结构体一共有三个：

* struct sched_class：cfs调度类的实现；
* cfs_rq：当前cpu使用cfs调度算法的进程队列；
* sched_entity：调度实体；

```c
/*
 * All the scheduling class methods:
 */
const struct sched_class fair_sched_class = {
	.next			= &idle_sched_class,
	.enqueue_task		= enqueue_task_fair,			// 将进程加入到cfs rq
	.dequeue_task		= dequeue_task_fair,			// 从cfs rq中删除
	.yield_task		= yield_task_fair,
	.yield_to_task		= yield_to_task_fair,
	.check_preempt_curr	= check_preempt_wakeup,
	.pick_next_task		= __pick_next_task_fair,		// 选择下一个进程
	.put_prev_task		= put_prev_task_fair,
	.set_next_task          = set_next_task_fair,
	.task_tick		= task_tick_fair,
	.task_fork		= task_fork_fair,					// 新进程创建，参数预处理，确定其vruntime
	.prio_changed		= prio_changed_fair,
	.switched_from		= switched_from_fair,
	.switched_to		= switched_to_fair,
	.get_rr_interval	= get_rr_interval_fair,
	.update_curr		= update_curr_fair,             // 更新进程的vruntime和更新cfsrq min_vruntime
};

struct cfs_rq {
	struct load_weight	load;							// 运行队列总的进程权重
	unsigned long		runnable_weight;
	unsigned int		nr_running;						// 进程的个数
	unsigned int		h_nr_running;      /* SCHED_{NORMAL,BATCH,IDLE} */
	unsigned int		idle_h_nr_running; /* SCHED_IDLE */
	u64			exec_clock;								// 运行的时钟
	u64			min_vruntime;							// 该cpu运行队列的vruntime推进值, 一般是红黑树中最小的vruntime值
	struct rb_root_cached	tasks_timeline;				// 红黑树的根结点，也能获取到left_rbmost
	struct sched_entity	*curr;							// 当前运行进程, 下一个将要调度的进程, 马上要抢占的进程, 
	struct sched_entity	*next;
	struct sched_entity	*last;
	struct sched_entity	*skip;
};

struct sched_entity {
	struct load_weight		load;				// 进程的权重
	unsigned long			runnable_weight;		
	struct rb_node			run_node;			// 运行队列中的红黑树结点
	struct list_head		group_node;			// 与组调度有关
	unsigned int			on_rq;				// 进程现在是否处于TASK_RUNNING状态

	u64				exec_start;					// 一个调度tick的开始时间
	u64				sum_exec_runtime;			// 进程从出生开始, 已经运行的实际时间
	u64				vruntime;					// 虚拟运行时间
	u64				prev_sum_exec_runtime;		// 本次调度之前, 进程已经运行的实际时间
};

```

### cfs调度关键参数

* 每个调度实体的vruntime：每次选择vruntime最小的调度实体去运行；
* cfs_rq的min_vruntime：该值是单调递增的，主要是用于计算新进程、从另外一个cpu迁移过来的进程和睡眠进程的vruntime的计算；

### cfs相关算法：物理时间和虚拟时间对应关系

$$
delta = delta_exec * \frac{NICE\_0\_LOAD}{curr->load.weight}
$$

* delta_exec：curr运行的实际物理时间；
* NICE_0_LOAD：nice值为0的权重，也就是1024；
* curr->load.weight：进程的权重；
* delta：进程虚拟时间的增加值


<!-- 分配给进程的运行时间 = 调度周期 * 进程权重 / 所有进程权重之和

vruntime = 实际运行时间 * 1024 / 进程权重

vruntime = (调度周期 * 进程权重 / 所有进程总权重) * 1024 / 进程权重 = 调度周期 * 1024 / 所有进程总权重  -->

### cfs调度基础

下面脑图描述了创建新进程后，经历的流程。特别需要注意`update_curr`和`place_entity`这两个函数，`update_curr`负责更新当前运行进程的vruntime和更新cfs rq的min_vruntime。当有新的进程加入该cfs rq时，`place_entity`计算该新进程合理的vruntime。结合脑图有以下几个问题：

1. 如何保证min_vruntime的单调递增？ min_vruntime在`update_curr`函数里被更新，可以知道min_vruntime = max(cfs_rq当前的min_vunrtime,cfs_rq curr的vruntime,cfs leftmost的vruntime)；
2. 在`task_for_fair`函数里，新进程的vruntime为什么要减去cfs_rq->min_vruntime呢？ 因为父进程和新进程可能是在不同的cpu上运行的，不同cpu的cfs rq的min_vruntime可能是不同的，所以可以等到将该进程部署到具体的cpu时，再加上对应的cfs_rq->min_vruntime；
3. 在`place_entity`函数中，如果是新进程，则其vruntime添加惩罚值（延迟跟踪机制），如果是睡眠进程唤醒的话，则减去一定的thresh（以便其尽快得到调度）；
4. 在`place_entity`函数中，为什么进程最终的vruntime取一个最大值？主要是考虑到睡眠进程唤醒vruntime的更新。假设一个进程睡眠1ms，如果thresh为3ms，则会导致该调度实体的vruntime回退。

![](https://gitee.com/chengshuyi/scripts/raw/master/img/20200815105430.png)


### cfs调度比较器

vruntime是unsigned，首先将二者vruntime相减，然后结果转换成signed，最后同0比较，便可以得到正确的大小关系。

```c
static inline int entity_before(struct sched_entity *a,
				struct sched_entity *b)
{
	return (s64)(a->vruntime - b->vruntime) < 0;
}
```

举个例子，当其中一个溢出的时候，判断是否能够得到正确的大小关系：

```c
#include <iostream>
#include <memory>
using namespace std;
int main()
{
    unsigned long long int vruntime1 = ULLONG_MAX;
    unsigned long long int vruntime2 = ULLONG_MAX;
    long long int res = ((vruntime1 + 5) - vruntime2);
    cout << res << endl;								// 5  正确
    res = (vruntime1 - (vruntime2 + 5));
    cout << res << endl;								// -5 正确
    return 0;
}
```



<!-- CFS的思想就是让每个调度实体（没有组调度的情形下就是进程，以后就说进程了）的vruntime互相追赶，而每个调度实体的vruntime增加速度不同，权重越大的增加的越慢，这样就能获得更多的cpu执行时间。

实际上是以vruntime-min_vruntime为key，是为了防止溢出，反

同时缓存树的最左侧节点，也就是vruntime最小的节点，这样可以迅速选中vruntime最小的进程。

运行队列的min_vruntime的作用就是处理溢出问题的。


通过考虑各个进程的相对权重，将一个延迟周期的时间在活动进程之间进行分配。对于由某个可
调度实体表示的给定进程，分配到的时间如下计算： -->