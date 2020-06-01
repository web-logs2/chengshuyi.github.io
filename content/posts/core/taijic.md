---
title: "Towards artificial general ub "
date: 2020-05-25T19:30:10+08:00
description: ""
draft: true
tags: [类脑芯片]
categories: [类脑芯片]
---

清华大学为神经网络芯片的编程提出了一种通用的方案，他们提出了一个编译器，将一个已经训练好的无约束的神经网络转换成一个符合给定硬件平台约束的网络，从而实现神经网络应用和目标硬件平台的解耦。其主要工作流程包括4个步骤，如下图。

![](https://gitee.com/chengshuyi/scripts/raw/master/img/20200601084808.png)



他们用计算图（computational graph）的概念来对神经网络进行优化和更改，然后对图进行重组，用类似于硬件核可以完成的一些基本操作来对图进行转化，之后进行图调整，用分治的办法逐层进行，使图里面的操作都是**硬件可以直接进行**的，这一步分成3个步骤，分别是数据重编码、完全展开，即用核可行操作代替类似于核可行的操作，然后是权值调整，这一小步又可以分为三步并且也是用分治的思想来完成。最后是将满足硬件约束的网络映射到目标硬件平台上。

**建立计算图（CG）**：根据输入的网络参数、拓扑结构以及训练集等信息建立计算图。如下图（a）所示；

**图重组**：用类似于硬件核可以完成的一些基本操作来对图进行转化，

* 将dot-product、add_bias和RELU操作转换成一种dot-like的操作
* 实现max pooling
* 多个dot-like代替剩余的操作

**图调整：**

​	数据重编码：解决数据精度问题，主要是浮点数

​	完全展开：将core-op-like的操作转换成core-op操作

​	权值调整：

	* free tuning： 权值的精度调整，以符合硬件约束
	* value range tuning：
	* Rounding Tuning：

**映射：**在此之前，已经创建了只含有core-op的网络，以及满足硬件的约束

![](https://gitee.com/chengshuyi/scripts/raw/master/img/20200601085209.png)

