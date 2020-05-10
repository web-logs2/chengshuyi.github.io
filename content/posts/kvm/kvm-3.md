---
title: "Kvm ARM论文三（Optimizing the Design and Implementation of the Linux ARM Hypervisor）"
date: 2020-04-06T14:08:58+08:00
description: ""
draft: true
tags: [kvm,嵌入式虚拟化,arm]
categories: [kvm,嵌入式虚拟化]
---





提出



### 简介

* 问题描述

从客户机内核切换到hypervisor内核需要上下文切换（使用的是同一套寄存器）。在X86中使用一条指令即可完成上下文的切换，而ARM并没有提供硬件机制，这就导致在上下文切换时需要保存和恢复相关寄存器。

* 解决方案：让Linux内核和hypervisor都运行在EL2。

之前，ARM架构上的KVM的hypervisor和它的OS内核运行在不同的CPU模式，即OS内核运行在EL1，而hypervisor运行在EL2。但是ARMv8.1新增了VHE（Virtualization Host Extensions），意味着我们不需要修改Linux代码就可以让Linux内核运行在EL2。这样做有三点优势：

1. 从VM切换到hypervisor不再需要保存EL1的寄存器；
2. hypervisor内核直接直接修改EL1的寄存器，不需要访问存放在内存的数据结构；
3. hypervisor和它的内核可以处于同一地址空间[^1]；

### 背景

介绍type-2类型的hypervisor研究现状在x86和ARM

* Intel VMX：VMX operations 和 EPT（Extended page tables）

VMX开启时，CPU有两种VMX operations：VMX root和VMX non-root。Root是Hypervisor运行环境，non-root是VM运行环境。**VMX root和non-root都独立拥有完整的CPU运行模式（用户和内核模式）【这点和ARM的不同】**。在non-root模式下执行敏感指令会陷入到root模式。hypervisor内核运行在root，VM内核运行在non-root，由于它们都运行在相同的CPU模式上，所以root和non-root必须提供复用。

1. VM Entry：
2. VM Exit：

保存在内存的数据结构VMCS

VMX root and non-root operation do not have separate CPU hardware modes,

* ARM VE

同intel VMX不同，ARM在原有的特权级上新增一个特权级EL2。EL1和EL2有很大的不同，比如运行在EL1的操作系统无法直接运行在EL2，因为EL2有自己的一套控制寄存器和单独的地址空间。因此hypervisor内核和VM内核都运行在EL1

* kvm

### hypervisor OS Kernel Support

[^1]: Third, the hypervisor and its OS kernel no longer need to operate across different CPU modes with separate address spaces which requires separate data structures and duplicated code. Instead, the hypervisor can directly leverage existing OS kernel functionality while at the same time configure ARM hardware virtualization features, leading to reduced code complexity and improved performance.