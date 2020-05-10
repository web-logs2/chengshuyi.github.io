---
title: "Kvm ARM论文一（KVM for ARM）"
date: 2020-04-05T15:10:53+08:00
description: ""
draft: true
tags: [kvm,嵌入式虚拟化,arm]
categories: [kvm,嵌入式虚拟化]
---

### 系统虚拟化：原理和实现

客户机操作系统所维护的页表负责传统的从客户机虚拟地址GVA到客户机物理地址 GPA的转换。如果MMU直接装载客户机操作系统所维护的页表来进行内存访问，那么由于页表中每项所记录的都是GPA，硬件无法正确通过多级页表来进行地址翻译。

针对这个问题，影子页表（Shadow Page Table）是一个有效的解决方法。如图4一10所 示，一份影子页表与一份客户机操作系统的页表对应，其作的是由GVA直接到HPA的地 址翻译





本文内容基于论文[^1]编写，所以部分内容可能与现在不符，比如ARM架构在2010年是不可虚拟化的，但是如今ARMv7和ARMV8都支持硬件虚拟化。

### 1. 简介

在ARM上做虚拟化的主要挑战是：ARM架构是不可虚拟化的，因为ARM的部分敏感指令运行在非特权模式下并不会产生trap，这样的话虚拟机管理程序将无法跟踪和维护当前虚拟机的状态。

据此，作者提出了基于KVM面向ARM架构的轻量级半虚拟化，轻量级半虚拟化是使用脚本自动修改客户机操作系统内核的源代码，将敏感非陷入指令替换成陷入指令，以此实现trap-and-emulate。更重要的是轻量级半虚拟化是架构相关的，只需要对指定架构编写替换脚本，该脚本则可用于该架构所有的操作系统。

### 2. 相关工作

* 1970s：虚拟化相关概念开始出现；

* 1990s：x86的硬件性能逐渐提高，提供了跑多个操作系统的硬件条件。由于运行在非特权模式下的敏感指令不会发生陷入，所以那时的x86还是不可虚拟化的。不过当时有以下三种方案可以跑多个操作系统：
  1. 全虚拟化：软件模拟客户机操作系统的每一条指令，缺点是太耗时；
  2. vmware提出了动态二进制翻译机制**（运行时检查每一条指令）**，即将敏感指令翻译成其它指令，缺点是过于复杂，实现难；
  3. 半虚拟化：xen提出半虚拟化，即**手动修改客户机操作系统源码**，用hypercall替换敏感指令，缺点工作量大，跟不上客户机操作系统版本更新；

* Intel和AMD提供硬件虚拟化分别是intel VT和AMD-V。
  1. 引入了新的cpu运行模式：客户机模式，在客户机模式下的敏感指令会自动陷入；
  2. 提供嵌套页表的硬件翻译；

* kvm利用了linux提供的机制，以linux模块的方式实现虚拟化，相较于vmware和xen要简单很多。但是却不支持arm，没有kvm arm的开源代码。

* xen-arm是唯一的开源ARM虚拟化：需要修改源码（大约4500行），但是提供的客户机Linux版本较老，2.6.11版本的linux内核；

* kvm-arm轻量级虚拟化：保存了kvm的特性，提共自动化脚本，该脚本自动修改linux源码。

#### 3. cpu虚拟化

>Popek and Goldberg define sensitive instructions as the group of instructions where the effect of their execution depends on the mode of the processor or the location of the instruction in physical memory.

1. 在不同的处理器模式下，指令的执行结果不一样；比如在用户模式下访问R13寄存器和特权模式下访问R13寄存器结果是不一样的；
2. 涉及到内存物理地址的指令，

敏感指令(Sensitive Instructions)的定義： **敏感指令是這樣一組指令，這些指令的行為取決於指令執行時處理器的工作模式，以及指令在記憶體中的位置。**

ARM处理器模式[^2]：

|       **处理器工作模式**        |          **特权模式**          |           **异常模式**           |                  **说明**                   |
| :-----------------------------: | :----------------------------: | :------------------------------: | :-----------------------------------------: |
|        用户（user）模式         |                                |                                  |              用户程序运行模式               |
|       系统（system）模式        | 该组模式下可以任意访问系统资源 |                                  |          运行特权级的操作系统任务           |
|       一般中断（IRQ）模式       | 该组模式下可以任意访问系统资源 | 通常由系统异常状态切换进该组模式 |                普通中断模式                 |
|       快速中断（FIQ）模式       | 该组模式下可以任意访问系统资源 | 通常由系统异常状态切换进该组模式 |                快速中断模式                 |
|     管理（supervisor）模式      | 该组模式下可以任意访问系统资源 | 通常由系统异常状态切换进该组模式 | 提供操作系统使用的一种保护模式，swi命令状态 |
|        中止（abort）模式        | 该组模式下可以任意访问系统资源 | 通常由系统异常状态切换进该组模式 |       虚拟内存管理和内存数据访问保护        |
| 未定义指令终止（undefined）模式 | 该组模式下可以任意访问系统资源 | 通常由系统异常状态切换进该组模式 |        支持通过软件仿真硬件的协处理         |

The sensitive privileged instructions defined by the ARM architecture are the coprocessor access instructions which are used to access the coprocessor interface.在一般寄存器和16个协处理器寄存器之间传送数据

15号协处理器：系统控制协处理器，控制系统的虚拟内存系统

14：用于访问浮点数寄存器

![](https://gitee.com/chengshuyi/scripts/raw/master/img/armasm_pge1464343210583.png)

 ARM体系结构还定义了敏感的非特权指令，这些指令涉及到：处理器模式相关寄存器、状态寄存器和内存访问指令。

* 处理器模式相关寄存器：参考上表，arm有七个处理器模式，即用户模式和六个特权级模式，每个模式下都对应一系列的banked寄存器。对这些寄存器访问的指令包括：LDM、STM。**由于客户机运行在用户模式下，所以需要借助VMM来获取或存储其它特权模式下对应的寄存器。所以LDM和STM部分指令需要进行修改。**
* 状态寄存器（CPSR、SPSR）：访问状态寄存器涉及到的指令：CPS、MRS、MSR、RFE和SRS
* 内存访问指令：

This instruction accesses memory as if the access were made by  user-mode,  even if actually in a privileged mode, and applies the  permission check  based on the code being "user". This is useful in a  kernel where a  user-space process passes a pointer to the kernel, and  you want to  ensure that the user process, not the kernel, had  permissions to read  the data.

This instruction could be used in a privileged code, such as an  exception handler, to test whether an access is possible in thread mode. For example, if a user mode access were aborted, the exception handler  may try to correct the problem by changing the memory protection  settings. The LDRT could then be used to test whether the access was now possible.

   http://infocenter.arm.com/help/index.jsp?topic=/com.arm.doc.faqs/ka14336.html

status register:CPSR SPSR     CPS MRS MSR RFE SRS

memory access

#### 轻量级半虚拟化

敏感非特权指令使用`SWI`指令替换，`SWI`指令有24bit的立即数可以用来重新编码敏感非特权指令。

使用协处理器相关指令，利用ARM的未定义指令异常实现陷入



ARM一共有24种敏感非特权指令，可以分成15类

`SWI`的高4位编码，剩余20位编码类型，5个状态寄存器访问指令，17位编码它的操作数；12个数据处理指令



The approach differs from Xen’s paravirtualization solution in that it requires no knowledge of how the guest is engineered and can be applied automatically on any OS source tree compiled by GCC. For instance, Xen defines a whole new file in arch/arm/mm/pgtbl-xen.c, which contains functions based on other Xen macros to issue hypercalls regarding memory management. Calls to these functions are placed instead of existing kernel code through the use of preprocessor conditionals many places in the kernel code. The presented solution completely maintains the original kernel logic, which drastically reduces the engineering cost and makes the solution more suitable for test and development of existing kernel code.

#### 异常

### 内存虚拟化

### 参考文献

[^1]:Dall, Christoffer & Nieh, Jason. (2010). KVM for ARM. Proceedings of the Ottawa Linux Symposium. 
[^2]: [ARM处理器的7种工作模式和2种工作状态](https://blog.csdn.net/ly930156123/article/details/79219303)





https://www.floobydust.com/virtualization/lawton_1999.txt A more complete list of bad instructions on IA32 can be found at





copy_from_user

```c
/*  * Test whether a block of memory is a valid user space address.
 * Returns 1 if the range is valid, 0 otherwise.
 *  * This is equivalent to the following test:  * (u65)addr + (u65)size <= current->addr_limit  *  * This needs 65-bit arithmetic.
 */ #define __range_ok(addr, size)						\ ({									\ 	unsigned long flag, roksum;					\ 	__chk_user_ptr(addr);						\ 	asm("adds %1, %1, %3; ccmp %1, %4, #2, cc; cset %0, ls"		\ 		: "=&r" (flag), "=&r" (roksum)				\ 		: "1" (addr), "Ir" (size),				\ 		  "r" (current_thread_info()->addr_limit)		\ 		: "cc");						\ 	flag;								\ })  #define access_ok(type, addr, size)	__range_ok(addr, size)
```

