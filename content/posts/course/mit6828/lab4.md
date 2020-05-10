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

### Multiprocessor Support and Cooperative Multitasking

1. 将设备物理地址映射到内核虚拟地址空间（参考lab3的虚拟地址空间分布图）；
2. 通过核间终端启动其它的CPU，并告诉该CPU第一条指令的首地址；
3. 设置相关`per-cpu`变量，包括：Per-CPU kernel stack、Per-CPU TSS and TSS descriptor、Per-CPU current environment pointer和Per-CPU system registers（每个cpu独有的寄存器）；
4. 实现大内核锁；
5. 实现Round-Robin Scheduling；
6. 利用系统调用实现在用户态创建一个新的进程，也就是`fork`的方式，jos有两种实现方式，一种是dump_fork实现的，即在fork时为新进程分配内存；另外一种是基于copy-on-write的方式，即将父进程和子进程的页表项置为COW，在修改该页表项对应的物理页时会触发page fault异常，有该异常处理函数分配具体内存；

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