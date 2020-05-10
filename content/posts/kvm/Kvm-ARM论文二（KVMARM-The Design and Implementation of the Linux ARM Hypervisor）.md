---
title: "Kvm ARM论文二（KVM/ARM: The Design and Implementation of the Linux ARM Hypervisor）"
date: 2020-04-06T14:08:56+08:00
description: ""
draft: true
tags: [kvm,嵌入式虚拟化,arm]
categories: [kvm,嵌入式虚拟化]
---

### 简介

1. 引入split-mode虚拟化，将hypervisor分成两部分，它们运行在cpu不同的模式上。ARM引入了的hyp模式，但是同内核模式并不相同，hyp模式设计之初只是为了运行hypervisor。而不是hosted hypervisor。运行在内核模式下的hypervisor可以使用Linux的特性，运行在hyp模式下的hypervisor可以使用hyp模式的特性。
2. 开源
3. 做实验
4. 帮助硬件设计

### ARM Virtualization Extensions

latest ARMv7和ARMv8

cortex-A15

* cpu虚拟化：在TrustZone实现hypervisor不太现实

![](https://gitee.com/chengshuyi/scripts/raw/master/img/20200407141634.png)

通过在hyp模式下配置相关寄存器，可以让执行在内核模式下的敏感指令陷入到hyp模式

* 内存虚拟化：IPA
* 中断虚拟化：可以配置陷入所有中断到hyp  gicv2引入vgic
* 时钟虚拟化：a new counter virtual counter and a new timer
* 同x86对比

ARM提供虚拟化是划分出一个新的cpu模式，即hyp模式。这个模式同用户和内核模式是不同的。相反，Intel则分为root和non-root，类似于ARM的Non-secure和Secure。**在root和non-root中都包含用户和内核模式，而且其功能完全相同。而ARM的hyp模式则是完全不同的、全新的模式，它用于一些独立的特性。**

Intel提供了保存/恢复客户机/hypervisor上下文硬件机制，只需要一条指令硬件自动保存/恢复上下文。ARM则不提供该硬件机制。

ARM and Intel are quite similar in their support for virtual

izing physical memory. Both introduce an additional set of page

tables to translate guest to host physical addresses. ARM ben

efited from hindsight in including Stage-2 translation whereas

Intel did not include its equivalent Extended Page Table (EPT)

support until its second generation virtualization hardware.



ARM’s support for virtual timers have no real x86 counterpart,

and until the recent introduction of Intel’s virtual APIC sup

port [20], ARM’s support for virtual interrupts also had no x86

counterpart. Without virtual APIC support, EOIing interrupts in

an x86 VM requires traps to root mode, whereas ARM’s virtual

GIC avoids the cost of trapping to Hyp mode for those interrupt

handling mechanisms. Executing similar timer functionality by

a guest OS on x86 will incur additional traps to root mode com

pared to the number of traps to Hyp mode required for ARM.

Reading a counter, however, is not a privileged operation on

x86 and does not trap, even without virtualization support in the

counter hardware.

### hypervisor架构

挑战：Hyp模式设计只考虑了type-1类型的虚拟化

#### split-mode virtualization

问题：

1. Linux代码只能运行在内核模式，不能够运行在hyp模式（ARM的hyp模式同内核模式并不是完全相同的）；需要修改大量的代码让linux运行在hyp模式；
2. 兼容性，当不适用虚拟化时
3. 及那个linux代码运行在hyp模式严重影响性能，hyp模式只提供了一个页表基址寄存器；内核模式使用两个，一个用于内核地址空间，一个用于用户地址空间
4. 这些问题不会发生x86,linux可以运行在root模式下的内核模式，也可以运行在non-root模式下的内核模式；



lowvisor和highvisor

lowvisor运行在hyp模式：1. 设置正确的运行上下。是能保护和隔离在；

2. 从vm切换到host
3. 提供虚拟化trap handler，处理中断和异常

![](https://gitee.com/chengshuyi/scripts/raw/master/img/20200407151030.png)

highvisor运行在内核模式

#### 3.3 内存虚拟化

stage-2 translation is disabled when running in the highvisor and lowviser, 只有当切换到VM时才会使能stage-2 translation



stage-2 page faults   IPA of the fault, 分配内存

### 4. Implementation and Adoption