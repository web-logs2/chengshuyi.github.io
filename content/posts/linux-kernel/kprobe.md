---
title: "linux Kprobe源码解析"
date: 2020-05-14T09:09:19+08:00
description: ""
draft: false
tags: [linux内核]
categories: [linux内核]
---

kprobe是linux内核的一个重要特性，是一个轻量级的内核调试工具，同时它又是其他一些更高级的内核调试工具（比如perf和systemtap）的“基础设施”。利用kprobe可以在内核任何代码位置插入用户代码，因此可以在一些关键点插入kprobe达到收集内核运行信息的目的。kprobe的机制也很简单，就是将被探测的位置的指令替换为断点指令（不考虑jmp优化），断点指令被执行后会通过notifier_call_chain机制来通知kprobes，kprobes会首先调用用户指定的pre_handler接口。执行pre_handler接口后会单步执行原始的指令，如果用户也指定了post_handler接口，会在调用post_handler接口后结束处理。基本的处理过程如下图所示：

![](https://gitee.com/chengshuyi/scripts/raw/master/img/20200814080244.png)

上图是x86平台下的流程，arm平台也大致差不多（除了触发断点的指令不同）。本文只涉及arm64平台的kprobe源码。



### kprobe结构体



kprobe结构体比较简单，只要注意两点即可：

1. `kprobe_opcode_t *addr`、`const char *symbol_name`和`unsigned int offset`三者的关系：`addr`单独使用、`symbol_name`和`offset`联合使用。因为`symbol_name`只能定位到函数的首地址，加上`offset`就可以定位到函数内部的任意一条指令；
2. `fault_handler`：




```c
typedef int (*kprobe_pre_handler_t) (struct kprobe *, struct pt_regs *);  //
typedef void (*kprobe_post_handler_t) (struct kprobe *, struct pt_regs *,
				       unsigned long flags);
typedef int (*kprobe_fault_handler_t) (struct kprobe *, struct pt_regs *,
				       int trapnr);
typedef int (*kretprobe_handler_t) (struct kretprobe_instance *,
				    struct pt_regs *);

struct kprobe {
	struct hlist_node hlist;                                        // 所有注册的kprobe都会添加到kprobe_table哈希表中
	struct list_head list;                                          // 如果在同一个位置注册了多个kprobe，这些kprobe会形成一个队列
	unsigned long nmissed;
	kprobe_opcode_t *addr;                                          // 探测的地址
	const char *symbol_name;                                        // 探测点的符号名称。名称和地址不能同时指定，否则注册时会返回EINVAL错误
	unsigned int offset;                                            // 探测点相对于符号地址的偏移，同symblo_name联合使用
	kprobe_pre_handler_t pre_handler;                               //
	kprobe_post_handler_t post_handler;                             //
	kprobe_fault_handler_t fault_handler;                           //
	kprobe_opcode_t opcode;                                         // 被修改的指令保存下来以便返回时恢复    
	struct arch_specific_insn ainsn;                                // 保存了探测点原始指令的拷贝。这里拷贝的指令要比opcode中存储的指令多，拷贝的大小为MAX_INSN_SIZE * sizeof(kprobe_opcode_t)。
	u32 flags;
};
```



### register_kprobe

前面讲到kprobe的基本结构，接着我们会看到kprobe的注册流程，下面的脑图已经很清晰了，见脑图：

![](https://gitee.com/chengshuyi/scripts/raw/master/img/20200811132549.png)



### breakpoint handler注册

前面讲到在某个内核地址或者符号注册一个kprobe时，kprobe会将该地址或者符号+offset对应的地址的指令保存下来，然后用触发断点指令替换掉，也就是BRK_OPCODE_KPROBES指令。接下来，我将会介绍到内核如何将断点中断路由给kprobe进行处理。

kprobe自身也是作为linux内核的一个模块而存在，linux内核启动时会加载并初始化每一个模块，kprobe也不例外。`kernel/kprobes.c`是初始化kprobe的源码，在`init_kprobes`会调用`arch_init_kprobes`，`arch_init_kprobes`是系统结构相关代码初始化，kprobe的breakpoint handler就是在这个函数内进行注册的。见下面的代码：

```c
int __init arch_init_kprobes(void)
{
	register_kernel_break_hook(&kprobes_break_hook);	// 注册断点kprobe处理函数
	register_kernel_step_hook(&kprobes_step_hook);		// 注册单步kprobe处理函数
	return 0;
}

static struct break_hook kprobes_break_hook = {
	.imm = KPROBES_BRK_IMM,								// 这里的立即数就是之前脑图上提到的16位立即数，用于标识kprobe
	.fn = kprobe_breakpoint_handler,
};

static struct step_hook kprobes_step_hook = {
	.fn = kprobe_single_step_handler,
};

```

先简述一下整个流程：

1. krpobe模块调用`arch_init_kprobes`；
2. `register_kernel_break_hook`：当内核出现断点中断时，内核会检查出现断点的位置的断点指令（同步中断的基础概念），根据断点指令的立即数同`break_hook`结构体的`imm`成员比较，匹配成功则调用回调函数，执行进一步的处理；
3. `register_kernel_step_hook`：同`break_hook`不同，`step_hook`没有立即数，但是kprobe会检查单步的指令地址来判断是否是自己发出的单步请求。


```c
struct break_hook {
	struct list_head node;
	int (*fn)(struct pt_regs *regs, unsigned int esr);
	u16 imm;                                                        // 这里的立即数就是之前脑图上提到的16位立即数，用于标识kprobe
	u16 mask; /* These bits are ignored when comparing with imm */
};

struct step_hook {
	struct list_head node;
	int (*fn)(struct pt_regs *regs, unsigned int esr);
};
```

下面的脑图详尽的描述了kprobe的`kprobe_breakpoint_handler`和`kprobe_single_step_handler`函数流程。需要注意以下两点：

1. kprobe对于post handler的触发做了优化，也就是判断探测地址的指令是否支持模拟，如果支持模拟直接进行模拟，省去了单步的开销；
2. 需要检查单步的请求是否是自己发出的，因为单步没有立即数用来区分是否是kprobe发出的单步。

![](https://gitee.com/chengshuyi/scripts/raw/master/img/20200814091000.png)

### 参考阅读

1. [An introduction to KProbes](https://lwn.net/Articles/132196/)