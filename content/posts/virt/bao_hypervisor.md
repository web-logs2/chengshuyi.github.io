---
title: "Bao_hypervisor"
date: 2020-05-29T21:29:29+08:00
description: ""
draft: true
tags: []
categories: []
---

**摘要**：随着嵌入式系统的快速发展：系统的复杂性和mixed-criticality，虚拟化可以实现较强的时空隔离。
jailhouse虽然可以解决嵌入式需求，但是jailhouse仍然依赖linux去启动和管理它的虚拟机。本文，作者提出了Bao hypervisor，

1. armv8和risc-v平台
2. 在尺寸、启动、性能和中断延迟方面只有很小的性能损耗

### 简介：

在自动驾驶和工业控制领域，相比于过去在功能需求方面有了较大的增长。随着复杂应用和计算消耗型应用的出现，对高性能的嵌入式开发环境的需求越来越高。
也导致从单核的微控制器（裸机应用，rtos）转移到多核平台，包含内存架构和GPOS.
同时，降低尺寸、重量、功耗和开销的市场压力导致若干个子系统需要运行在同一个硬件平台。
而且，一般采用MCS的形实，集成多个不同特权等级的模块。比如，自动驾驶系统中网络互联娱乐系统一般部署在非安全的控制系统。
所以，对于mixed-criticallity需要平衡这些安全

虚拟化是一个很成熟的技术


静态分区的hypervisor以jialhouse为主（很小的软件层），它假设虚拟机之间不共享任何资源，所以
1. 不需要调度器，静态分配cpu
2. 不需要于一服务，进一步降低系统的复杂性
3。 很强的安全
但是jailhouse仍然依赖于linux去启动和管理cells，承受着之前所讲的和其他的hypervisor类似的安全问题。

尽管在cpu和内存方面有很强的隔离性，但是仍然有一些属于共享资源，比如
1. last-level cache
2. 互联总线
3. 内存控制器
可以导致Dos Attack（一个虚拟机使用大量的共享资源）
解决这个问题：cache划分

虽然Bao和Jailhouse的架构相同，但是bao不依赖于外部依赖，（除了固件，操作底层管理）。提供了cache着色，


BAo hypervisor

bao时secruity和safety的轻量级罗金属Hypervisor。
只包含一小部分的软件曾，特权指令支持静态划分
内存划分通过2-stage转换实现
IO支支持直通
虚拟中断直接映射到物理cpu，不需要调度器。
提供了基于静态共享内存划分的inter-vm通信机制
inter-vm的异步通知机制通过触发Hypercall来实现

除了平台管理的固件之外，Bao没有外部依赖


平台支持

BAO除了串口驱动，没有别的驱动了，所以只需要很小的改动就可以移植到别的平台
在arm平台下，依赖于PSCI去执行底层的电源控制操作，进一步降低了平台相关的驱动
这些都已经ATF提供
linux依赖于PSCI去支持cpu的热拔插，当客户机调用psci服务时，bao仅仅检查参数，保证vm的隔离
尽管bao可以直接从ATF启动，但是仍然使用uboot去加载hypervisor和客户机镜像

时空隔离

### 3. 实验

本节是关于bao hypervisor的性能指标，主要涉及三个大指标，分别是代码体积和占用内存、启动时间和中断时延

#### 3.1 代码体积和内存占用

#### 3.2 启动性能

#### 3.3 性能损耗和干扰

#### 3.4 中断时延



### 4. 相关工作

虚拟化技术起源于1970年代，就技术本身而言已经很成熟了。

xen和kvm在开源的hypervisor中属于佼佼者。xen是裸金属hypervisor，它包括一个特权的VM（DOM0）和其它非特权的VM（DOMU），DOM0管理DOMU和外设接口。kvm的设计理念则完全不同，kvm运行在宿主机上。

* xen on arm：xen在arm平台上的实现；

* rt-xen：实时调度的xen；

* kvm/arm：提出split-mode虚拟化，推进arm平台下硬件虚拟化；

基于已经开发的微内核实现虚拟化，提供较强的实时性；

基于arm平台下的trustzone，提供虚拟化功能；

一系列小型的type-1类型的hypervisor：

* Xtratum：面向LEON处理器，强调安全性；
* Hellfire/prplHypervisor：面向MIPS架构，强调实时性；
* XVisor：我记得是全虚拟化的，支持imx6；
* ACRN：面向x86，强调IOT，也支持aliosthings；
* Minos：面向armv8；
* Bao：面向arm和risc-v，采用静态分区策略；

西门子的jailhouse在静态分区虚拟化实现上是领先者，但是依赖于linux系统，自身作为linux的模块；

谷歌的Hafnium重视安全，type-1类型的hypervisor，旨在提供内存隔离，将可信代码和非可信代码区分开。

### 5. On the road

目前，作者正在开发更多的特性，如SMMU和GIC（V3和V4）。支持更多的平台，如TX2和IMX8。同时，希望能够重构代码以满足MISRA C语言规范。

？

？

### 6. 结语

Bao hypervisor是静态分区架构的hypervisor。尽管仍处于研发初期，但是其ROADMAP指出，未来会支持更多的平台以及每个分区系统的TEE。



