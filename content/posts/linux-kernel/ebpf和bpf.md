---
title: "Ebpf和bpf"
date: 2020-05-11T17:29:09+08:00
description: ""
draft: true
tags: []
categories: []
---

### BPF: the universal in-kernel virtual machine [LWN.net]

3.15版本bpf出现两个分支，一个是cBPF和internal BPF

1. 可以使用的寄存器从2到10
2. 增加了指令集，更符合真实的硬件指令；使得能够调用内核函数，或者其它的更多功能；



secure computing和packet filtering仍然可以接收古老的bpf程式，但是会被转换成internal bpf程序的格式



在3.16版本internal BPF也支持了jit compiler

bpf使用范围已经不仅仅局限于网络系统，

### **Extending extended BPF**

https://lwn.net/Articles/604043/ 将bpf移动到kernel目录下，bpf可以使用在更多的场合



ebpf增加了若干特性和性能上的提升



patch添加了新的系统调用bpf以及相关用户库

int bpf(BPF_PROG_LOAD, int prog_id, enum bpf_prog_type type, struct nlattr *prog, int len);

int bpf_prog_load(int prog_id, enum bpf_prog_type prog_type, 

struct sock_filter_int *insns, int prog_len, const char *license);



verifier检查程序是否有问题

 It tracks the state of every eBPF register and will

not allow their values to be read if they have not been set. To the extent possible, the type of

the value stored in each register is also tracked. Load and store instructions can only operate

with registers containing the right type of data (a "context" pointer, for example), and all

operations are bounds-checked. The verifier also disallows any program containing loops,

thus ensuring that all programs will terminate.





 if that license is not GPL-compatible, the kernel

will refuse to load the program. This behavior already appears to be somewhat controversial;





maps



### **A reworked BPF API**





### ebpf简介

位于用户空间中的应用在内核中辟出一块空间建立起一个数据库用以和 eBPF  程序交互(bpf_create_map())；

map 带来的最大优势是效率：相对于 cBPF 一言不合就把一个通信报文从内核空间丢出来的豪放，map 机制下的通讯耗费就要小家碧玉的多了：还是以 sockex1 为例，一次通信从内核中仅仅复制 4个字节，而且还是已经处理好了可以直接拿来就用的，

map 机制解决的另一个问题是通信数据的多样性问题。





Using eBPF maps is a method to keep state between invocations of the eBPF program, and allows sharing data between eBPF kernel programs, and also between kernel and user-space applications.

共享数据在ebpf内核程序

内核和用户程序

![](https://gitee.com/chengshuyi/scripts/raw/master/img/20200512085617.png)

```
bpf(BPF_MAP_CREATE, &bpf_attr, sizeof(bpf_attr));
```





### Kernel analysis using eBPF

eBPF in its traditional use case is attached to networking hooks allowing it to filter and classify 

network traffic using (almost) arbitrarily complex programs



Furthermore, we can attach eBPF programs to tracepoints, kprobes, and perf events for 

debugging the kernel and carrying out performance analysis.



10个通用的64位寄存器，一个frame pointer寄存器

每个指令都是64位，最多包含4096个指令



alu指令

内存指令

跳转指令

BPF_call指令可以调用内核函数

![](https://gitee.com/chengshuyi/scripts/raw/master/img/20200512154411.png)



jit编译器：寄存器对应情况

![](https://gitee.com/chengshuyi/scripts/raw/master/img/20200512154552.png)



verifier

检查程序是否能够正常运行

检查license

检查函数调用

检查程序的控制流程图（有向无环图）

map:

内核和用户程序之间传递数据

通过fd进行管理map：create lookup update delete

