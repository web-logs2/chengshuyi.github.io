---
title: "Lab3 - 实验笔记"
date: 2020-04-25T13:34:13+08:00
description: ""
draft: false
tags: [mit6828 OS实验]
categories: [mit6828 OS实验]
---

### User Environments and Exception Handling

* `env_init`函数的实现类似于`page_init`函数，主要将空闲的`struct Env`插入到`env_free_list`链表上；
* `env_setup_vm`函数为该进程分配一个页目录表，并将内核页目录表内容拷贝到该进程的页目录表；
* `region_alloc`函数为进程分配内存，并将其映射到其地址空间；
* `load_icode`函数解析elf文件头部信息，将其加载到指定的地址上；修改进程的eip值；为该进程创建用户栈；
* `env_alloc`函数工作流程：调用`env_setup_vm`完成进程的页目录表的分配；分配一个id；设置该进程的寄存器的值，包括：ds、es、ss、esp、cs等；
* `env_create`函数先后调用了`env_alloc`函数和`load_icode`函数；
* `env_run`函数根据进程是否发生切换来决定是否更新cr3寄存器的值；

中断流程如下：

1. 中断或者异常发生，cpu判断当前特权级来决定是否切换到内核栈；
2. 如下图所示，如果发生特权级切换需要特别保存ss和esp寄存器（中断嵌套的话只能发生在内核态）；
3. 保存剩余的寄存器到内核栈中（具体可以查看int指令执行的具体内容）；
4. 根据中段号跳转到IDT的相应entry；
5. 压入中断号；
6. 保存用户上下文；
7. 中断处理函数；
8. 恢复用户上下文；
9. 执行iret指令，从内核栈中pop出ss和esp寄存器；

![](https://gitee.com/chengshuyi/scripts/raw/master/img/20200425200239.png)

### Page Faults, Breakpoints Exceptions, and System Calls

page falults比较简单，不再叙述；

breakpoints exceptions：貌似linux上的gdb调试程序就是通过这个异常来实现的；

系统调用：这里需要注意的是当前不再通过栈传递参数，而是通过寄存器传递，系统调用号存在eax寄存器，剩余的五个参数分别对应着 %edx, %ecx, %ebx, edi, 和esi。返回值存放在eax寄存器；

#### User-mode startup

初始化该进程的环境变量，比如thisenv。

#### Page faults and memory protection

用户进程传进来的指针一定要检查，首先检查该指针不能在内核地址空间，然后判断是否存在该虚拟地址，最后检查读写权限；

