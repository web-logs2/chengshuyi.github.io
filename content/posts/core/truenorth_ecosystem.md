---
title: "Truenorth Ecosystem for Brain-Inspired Computing: Scalable Systems, Software, and Apllications笔记"
date: 2020-06-07T19:36:13+08:00
description: ""
draft: true
tags: [类脑芯片]
categories: [类脑芯片,论文笔记]
---

**摘要：**本文主要描述了基于类脑芯片-TrueNorth（功耗为75mW、1m个神经元、256m个突触）芯片组成的大规模分布式的芯片（4096个芯片）集群。该集群主要有：NS1e、NS1e-16（水平扩展）和NS16e（垂直扩展）。软件方面提供了模拟器、编程语言、集成开发环境、算法库、深度学习相关的工具、教学资料和远程方案。对于scale-up系统来说主要是要降低芯片间和芯片内部的网络通信量。整个生态环境使用着超过30多个机构。truenorth芯片在这些应用中起着关键作用。

### 1. 简介

神经网络目前处于突飞猛进的状态，特别是深度学习。而且已经在很多任务有着很好的效果，比如：图片分类、语音识别、NLP等。从功耗方面来看，神经突触架构的芯片，如TrueNorth芯片，功耗和实时性相比于传统的架构有着很大的提升。特别地，深度学习的算法经过适当的修改便可以直接在类脑芯片进行训练，而且可以得到不错的精度。

芯片已经有了，但是仍然缺少丰富的软件工具，不能够充分的发挥出硬件平台的性能。因此，本文主要提出了一个新的应用于类脑芯片上的软件生态系统。该系统包括：硬件平台、无缝对接的软件工具、语言、库、环境和算法。所有的这些，可以让用户很方便的开发和部署相关应用。

![](https://gitee.com/chengshuyi/scripts/raw/master/img/20200608135611.png)

本文主要贡献有：

* NS1e-16：包含了16个芯片的NS1e平台；NS16e包含了4x4阵列的芯片；
* 描述了整个开发工作流程，包括：网络训练算法、库、模拟器、数据预处理、模型编译和其它的若干个软件工具；
* 提供了**高效的且简单的映射算法**，将逻辑网络映射到物理硬件上，降低了网络通信量（重点关注）；
* 向读者展示了我们强大的开发团队，提供了我们当前开发状态；

我们的软件大大的提高了生产力。

### 2. TRUENORTH

truenorth的芯片的主要思想源于神经科学，该芯片的特点是：并行、事件驱动、非冯诺依曼存储架构。

truenorth芯片的扩展性主要是它的modular tiled-core结构。truenorth的每一个神经突触核都代表着一个全连接的神经网络（256个输出神经元和256个输入轴突，256x256个突触）。。。？

truenorth芯片包含了4096个核心，64x64的阵列。所以，每个芯片有1m个神经元和256m个突触。所有的这些是通过三星的28nm LPP处理技术和5.4b个半导体。当用0.755v供电和运行视觉应用程序时，芯片只消耗大约75mW。。。。？

### 3. 硬件系统

本节主要介绍硬件平台，硬件平台分为几个基本组件，

#### A. NS1e平台

NS1e（the **N**eurosynaptic **S**ystem, **1** million neuron **e**valuation platform）

### 4. 软件系统

specifying、训练、编译和部署

![](https://gitee.com/chengshuyi/scripts/raw/master/img/20200613131649.png)

#### A. 工作流程

a. 数据采集（文件数据、usb或者以太网传感器、脉冲）；

b. 数据预处理，提取特征；

c. 特征被编码成脉冲；

d. turenorth芯片处理脉冲信号；

e. 处理芯片输出的脉冲，将其转换成有意义的数据；

f. 将数据传给应用程序；

#### B. 应用设计流程：简介

g-h. 训练阶段，采用DNN；

i-k. build阶段，采用corelet编程语言建立truenorth模型，用compass模拟器进行模型测试；

#### C. 应用设计流程：训练阶段

1）数据集：

#### D. 应用设计流程：build阶段

### 5. 应用

### 6. NS1E和NS16E的功耗测试

![](https://gitee.com/chengshuyi/scripts/raw/master/img/20200613130501.png)

### 7. 相关工作

 近年来，有几个小组提出了一种高效的硬件来模拟生物硬件。 neurogrid采用数模混合的方式使用了16个芯片模拟了1m个神经元，采用片外存储的方式存储连接参数。SpiNNaker... BrainScaleS...

为了进行粗略的比较，我们在五种数字计算体系结构分别运行深度神经网络并进行性能上的分析。单个truenorth芯片每秒可以处理1200-2600张32x32大小的彩色图片，其功耗大概是170-275mW，也就是6100-7350FPS/W。当前的多芯片的truenorth每秒可以处理430-1330张32x32大小的彩色图片，其功耗大概为0.89-1.5W，也就是360-1420FPS/W。SpiNNaker处理28x28的灰度图像大概是167FPS/W，Tegra K1 GPU、Titan X GPU核core i7 cpu处理224x224彩色图像的功耗分别是45FPS/W、14.2FPS/W和3.9FPS/W。

### 8. 未来展望

对于动态可扩展系统架构来说，涉及到核间、芯片间和板间的脉冲通信。在一个核内采用synaptic crossbar的方式实现通信；在核间采用片上网络的方式；板间的话采用以太网、Infifiniband或者PCIe。



<!--This scalable and hierarchical routing architecture is possible due to the exponential decay in hop distance and bandwidth observed in biological networks [14]. More importantly, this distribution is similar to that of the hop-length histograms of placed deep networks that are run on TrueNorth.-->

<!--[14]Network architecture of the long-distance pathways in the macaque brain-->

