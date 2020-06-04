---
title: "Lab4 - 实验笔记"
date: 2020-04-25T13:34:16+08:00
description: ""
draft: false
tags: [mit6828 OS实验]
categories: [mit6828 OS实验]
---

[lab4 实验代码：不过最后primes案例没过](https://github.com/chengshuyi/jos-lab/commit/287d90ad016d0dbb6d1d275336c4007cc5c14931)

[lab4实验代码patch1：通过primes案例](https://github.com/chengshuyi/jos-lab/commit/d108ac413223571d2417293fe5ce66bda8ca3920)

[lab4实验代码patch2：小bug](https://github.com/chengshuyi/jos-lab/commit/77e886ae1aeb99c42b604ec0b40bd9e05eb291a0)

### x86多核系统

JOS支持的是对称多处理器架构，最先启动的核称为bootstrap处理器（BSP），后续启动的核称为application处理器（APs）。下面介绍一下整个的启动流程：

1. `mp_init`函数：从BIOS中找到多处理器信息配置表；

2. `boot_aps`函数：

   a. 将APs的启动代码移动到0x7000的位置；

   b. 发送核间中断，参数包括：cpu编号和cpu起始运行地址；

   c. 等待该cpu启动完毕；

3. APs执行的代码和BSP差不太多，主要涉及的文件有：`kern/mpentry.S`和`kern/init.c`。

### per-cpu

per-cpu表示的是为每个cpu维护自己的数据结构，其次访问cpu共享的数据结构时需要加锁。

### Multiprocessor Support and Cooperative Multitasking

1. 将设备物理地址映射到内核虚拟地址空间（参考lab3的虚拟地址空间分布图）；
2. 通过核间终端启动其它的CPU，并告诉该CPU第一条指令的首地址；
3. 设置相关`per-cpu`变量，包括：Per-CPU kernel stack、Per-CPU TSS and TSS descriptor、Per-CPU current environment pointer和Per-CPU system registers（每个cpu独有的寄存器）；
4. 实现大内核锁；
5. 实现Round-Robin Scheduling；
6. 利用系统调用实现在用户态创建一个新的进程，也就是`fork`的方式，jos有两种实现方式，一种是dump_fork实现的，即在fork时为新进程分配内存；另外一种是基于copy-on-write的方式，即将父进程和子进程的页表项置为COW，在修改该页表项对应的物理页时会触发page fault异常，有该异常处理函数分配具体内存；

> **Exercise 1.** Implement `mmio_map_region` in `kern/pmap.c`. To see how this is used, look at the beginning of `lapic_init` in `kern/lapic.c`.

```c
void *mmio_map_region(physaddr_t pa, size_t size)
{
	static uintptr_t base = MMIOBASE;
    // 大小取整
	size = ROUNDUP(size,PGSIZE);
	if(size > MMIOLIM) panic("error");
    // 做映射
	boot_map_region(kern_pgdir,base,size,pa,PTE_PCD|PTE_PWT|PTE_W|PTE_P);
	base += size;
	return (void *)(base-size);
}
```

> **Exercise 2.** Read `boot_aps()` and `mp_main()` in `kern/init.c`, and the assembly code in `kern/mpentry.S`. Make sure you understand the control flow transfer during the bootstrap of APs. Then modify your implementation of `page_init()` in `kern/pmap.c` to avoid adding the page at `MPENTRY_PADDR` to the free list, so that we can safely copy and run AP bootstrap code at that physical address. 

```c
	extern char mpentry_start[];
	extern char mpentry_end[];
	s = MPENTRY_PADDR;
	e = mpentry_end-mpentry_start+s;
	s = ROUNDDOWN(s,PGSIZE);
	e = ROUNDUP(e,PGSIZE);
	s = PGNUM(s);
	e = PGNUM(e);
	for(i=s;i<e;i++){
		pages[i].pp_ref = 1;
		pages[i].pp_link = NULL;
	}
```

> **Exercise 3.** Modify `mem_init_mp()` (in `kern/pmap.c`) to map per-CPU stacks starting at `KSTACKTOP`, as shown in `inc/memlayout.h`. The size of each stack is `KSTKSIZE` bytes plus `KSTKGAP` bytes of unmapped guard pages.

```c
// 将每个cpu的内核栈映射到指定的虚拟地址上
static void mem_init_mp(void)
{
	int i;
	uintptr_t s = KSTACKTOP;
	for(i=0;i<NCPU;i++){
		boot_map_region(kern_pgdir,s-KSTKSIZE,KSTKSIZE,PADDR(percpu_kstacks[i]),PTE_W|PTE_P);
		s -= KSTKSIZE+KSTKGAP;
	}
}
```

> **Exercise 4.** The code in `trap_init_percpu()` (`kern/trap.c`) initializes the TSS and TSS descriptor for the BSP.

```c
void trap_init_percpu(void)
{
	// Setup a TSS so that we get the right stack
	// when we trap to the kernel.
	int i;
	thiscpu->cpu_ts.ts_esp0 = KSTACKTOP;
	for(i=0;i<thiscpu->cpu_id;i++){
		thiscpu->cpu_ts.ts_esp0 -= (KSTKSIZE+KSTKGAP);
	}
	thiscpu->cpu_ts.ts_ss0 = GD_KD;
	thiscpu->cpu_ts.ts_iomb = sizeof(struct Taskstate);

	// Initialize the TSS slot of the gdt.
	gdt[(GD_TSS0 >> 3) + thiscpu->cpu_id] = SEG16(STS_T32A, (uint32_t) (&thiscpu->cpu_ts),
					sizeof(struct Taskstate) - 1, 0);
	gdt[(GD_TSS0 >> 3) + thiscpu->cpu_id].sd_s = 0;

	// Load the TSS selector (like other segment selectors, the
	// bottom three bits are special; we leave them 0)
	ltr(GD_TSS0+(i<<3));

	// Load the IDT
	lidt(&idt_pd);
}
```



### Copy-on-Write Fork

下面是用户空间出现页异常处理流程：

1. 通过系统调用注册进程自己的处理函数（在fork的时候配置的）；
2. 当页异常触发时，首先由内核异常处理函数处理，判断该错误地址是否发生在用户地址空间。如果是的话则将进程的esp指向进程异常栈；将错误信息保存在进程异常栈（在fork的时候分配的内存）里；将返回值指向进程异常处理函数；
3. 返回用户空间运行。

下面是fork的流程：

1. 通过系统调用注册进程自己的处理函数；
2. 通过系统调用创建子进程；
3. 利用自映射机制可以快速遍历页表项，查看每个页表项的属性（低12位），如果该页存在的话，则将其映射到子进程的地址空间，并将其页表项型属性置为COW；然后修改自身的页表项的属性；（因为子进程和父进程都不能直接修改该页表项对应的物理页（原物理页），只能新建物理页并元物理页的内容拷贝进去。）
4. 为子进程分配进程异常栈；
5. 为子进程注册进程异常处理函数；（因为子进程运行的时候大部分的页属性是COW，也就是不可能自己通过系统调用注册进程异常处理函数）；
6. 修改子进程运行状态-》可运行状态。

### Preemptive Multitasking and Inter-Process communication (IPC)

实现时钟中断，利用时钟中断完成任务的抢占；

实现ipc，其ipc有如下机制：

1. ipc是基于系统调用实现的，可以传递一个整数或者传递一个虚拟地址（对应着一个物理页）；
2. ipc的发送方处于polling状态；
3. ipc的接收方处于进程不可运行态，由ipc的发送方不断的尝试发送数据，发送成功后由发送方修改接收方的进程运行状态；