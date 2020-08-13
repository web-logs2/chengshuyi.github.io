---
title: "Kprobe"
date: 2020-05-14T09:09:19+08:00
description: ""
draft: true
tags: [linux内核]
categories: [linux内核]
---

首先kprobe是最基本的探测方式，是实现后两种的基础，它可以在任意的位置放置探测点（就连函数内部的某条指令处也可以），它提供了探测点的调用前、调用后和内存访问出错3种回调方式，分别是pre_handler、post_handler和fault_handler，其中pre_handler函数将在被探测指令被执行前回调，post_handler会在被探测指令执行完毕后回调（注意不是被探测函数），fault_handler会在内存访问出错时被调用；jprobe基于kprobe实现，它用于获取被探测函数的入参值；最后kretprobe从名字种就可以看出其用途了，它同样基于kprobe实现，用于获取被探测函数的返回值。
————————————————
版权声明：本文为CSDN博主「luckyapple1028」的原创文章，遵循CC 4.0 BY-SA版权协议，转载请附上原文出处链接及本声明。
原文链接：https://blog.csdn.net/luckyapple1028/article/details/52972315


```c

typedef int (*kprobe_pre_handler_t) (struct kprobe *, struct pt_regs *);
typedef void (*kprobe_post_handler_t) (struct kprobe *, struct pt_regs *,
				       unsigned long flags);
typedef int (*kprobe_fault_handler_t) (struct kprobe *, struct pt_regs *,
				       int trapnr);
typedef int (*kretprobe_handler_t) (struct kretprobe_instance *,
				    struct pt_regs *);

struct kprobe {
	struct hlist_node hlist;                                        // 所有注册的kprobe都会添加到kprobe_table哈希表中
	/* list of kprobes for multi-handler support */
	struct list_head list;                                          // 如果在同一个位置注册了多个kprobe，这些kprobe会形成一个队列
	/*count the number of times this probe was temporarily disarmed */
	unsigned long nmissed;
	/* location of the probe point */
	kprobe_opcode_t *addr;                                          // 探测的地址
	/* Allow user to indicate symbol name of the probe point */
	const char *symbol_name;                                        // 探测点的符号名称。名称和地址不能同时指定，否则注册时会返回EINVAL错误
	/* Offset into the symbol */
	unsigned int offset;                                            // 探测点相对于addr地址的偏移
	/* Called before addr is executed. */
	kprobe_pre_handler_t pre_handler;                               //
	/* Called after addr is executed, unless... */
	kprobe_post_handler_t post_handler;                             //
	/*
	 * ... called if executing addr causes a fault (eg. page fault).
	 * Return 1 if it handled fault, otherwise kernel will see it.
	 */
	kprobe_fault_handler_t fault_handler;                           //
	kprobe_opcode_t opcode;                                         // 被修改的指令保存下来以便返回时恢复    
	struct arch_specific_insn ainsn;                                // 保存了探测点原始指令的拷贝。这里拷贝的指令要比opcode中存储的指令多，拷贝的大小为MAX_INSN_SIZE * sizeof(kprobe_opcode_t)。
	/*
	 * Indicates various status flags.
	 * Protected by kprobe_mutex after this kprobe is registered.
	 */
	u32 flags;
};
```

![](https://gitee.com/chengshuyi/scripts/raw/master/img/20200811132549.png)


```c
struct break_hook {
	struct list_head node;
	int (*fn)(struct pt_regs *regs, unsigned int esr);
	u16 imm;                                                        // 这里的立即数就是之前脑图上提到的16位立即数，用于标识kprobe
	u16 mask; /* These bits are ignored when comparing with imm */
};
```

```c
struct step_hook {
	struct list_head node;
	int (*fn)(struct pt_regs *regs, unsigned int esr);
};
```

![](https://gitee.com/chengshuyi/scripts/raw/master/img/20200811140405.png)