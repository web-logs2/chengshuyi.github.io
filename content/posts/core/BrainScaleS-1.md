---
title: "The Operating System of the Neuromorphic BrainScaleS-1 System论文笔记"
date: 2020-05-28T17:09:34+08:00
description: ""
draft: true
tags: [类脑芯片]
categories: [类脑芯片]
---

### 1. 简介

1. 作者指出当前的类脑架构对软件的需求：系统控制、数据预处理、数据交换和数据分析
2. 当前类脑系统都是由独立研究者开发



第一部分介绍硬件，

第二部分介绍软件

#### A. BrainScaleS-1 类脑系统

它[^1]使用VLSI（Very Large Scale Integration）实现电子电路模拟生物神经系统。当今的系统采用混合信号技术使得系统更具灵活性。最近，纯数字系统的出现，对比于仿生方法的优点是：确定性和可编程神经元的动态性。

基于脉冲的单芯片[^2]，BSS-1一直在进化，速度比生物计算快1000-10000

BSS-1包含384个芯片、每个芯片包含512个AdEx神经元和113k可塑性突触。xilinx fpga提供io接口、simulus和存储数据。fpga和控制集群网络通过1g和10g网络链接。

#### B. 进行试验

前任[^3]通过python来进行配置和实验。而BSS os使用python提供用户API，核心软件都是通过C++编写的。

PyNN模拟器[^4][^5]

BSS-1大约有50MB的参数需要进行配置，配置需要硬件的专家知识，BSS os则是消除该问题，提供易于使用的接口和非专家用户的支持

#### C. 实验平台

需要的技术有：资源管理、时分、公平、账户、监控和可视化

对于外部用户：鲁棒性、实验的可重复性和技术支持

### 2. 方法和工具

BSS os的主要目标是提供在BSS-1上易于操作的实验环境，包括：数据流和控制流。提供脉冲神经网络的API，比如PyNEST或者PyNN模拟器主要集中于初始实验环境的配置（网络拓扑、模型参数、可塑性规则、recording settings和stimulus definition）。同样的，硬件也需要相应的初始化配置

1. 压缩领域知识，将其转换成软件层
2. 参数配置的可靠性检查
3. 错误处理和反馈
4. api和硬件的一致性
5. 工具的可靠性
6. 软件各层的灵活性，可以让用户在每一层做适当的修改

BSS的软件开发已经持续了十多年（2008年开始），而且一直在持续更新和优化

#### A. 方法论

相比之前的工作，现如今引入了大规模的类脑系统，引入了更多的复杂性。比如说：多芯片的自动化配置和运行时控制。开环环境和流程的标准化：

1. 人多
2. 代码多
3. 开发周期长
4. 开源



开源

软件设计

review

验证

软件环境

#### B. 基础foundations

1. PyNN：网络拓扑、模型参数和可塑性规则、输入刺激的定义和recodring
2. 编程环境：操作系统是Linux，所有的核心库用c++编写（只有少部分和linux对接的部分用c语言编写）
3. python wrapping：脚本语言比编译语言更有优势、科学库、使用python wrapping链接python和C++[^6][^7]
4. 序列化：将内存数据变成可存储的格式，第三方库：boost::serialization（引用和指针）
5. 工具库：
   1. ranged enumeration type：c++不支持数据范围检查 https://github.com/ignatz/rant
   2.  *Python-style C++ convenience library*：用python血c++
   3. *Bit Manipulation Library*





![image-20200529142658580](C:\Users\dell\AppData\Roaming\Typora\typora-user-images\image-20200529142658580.png)

### 3. 实现

#### D. 网络描述 



### 5. 应用



 



[^1]: C. A. Mead. “Neuromorphic Electronic Systems”. In: Proceedings of the IEEE* 78 (1990), pp. 1629–1636.
[^2]: Schemmel, A. Grübl, K. Meier, et al. “Implementing Synaptic Plasticity in a VLSI Spiking Neural Network Model”. In: *Proceedings of the 2006 International Joint*Conference on Neural Networks (IJCNN)*. IEEE Press, 2006.
[^3]:Daniel Brüderle, Eric Müller, Andrew Davison, et al. “Establishing a novel modeling tool: a python-based interface for a neuromorphic hardware system”. In: Frontiers in Neuroinformatics* 3 (2009), p. 17. issn: 1662- \5196. doi: 10.3389/neuro.11.017.2009. url: https://www. frontiersin.org/article/10.3389/neuro.11.017.2009.
[^4]:A. P. Davison, D. Brüderle, J. Eppler, et al. “PyNN: a common interface for neuronal network simulators”. In: *Front. Neuroinform.* 2.11 (2009). doi: 3389/neuro.11. 011.2008
[^5]: Oliver Rhodes, Petruţ A. Bogdan, Christian Brenninkmeijer, et al. “sPyNNaker: A Software Package for Running PyNN Simulations on SpiNNaker”. In: Frontiers in Neuroscience* 12 (2018), p. 816. issn: 1662- 453X. doi: 10.3389/fnins.2018.00816. url: https://www.frontiersin.org/article/10.3389/fnins.2018.00816.
[^6]: Roman Yakovenko. *pygccxml/py++*. url: https : / /sourceforge.net/projects/pygccxml.
[^7]:Boost.Python. *Version 1.71.0 Website*. http://www.boost.org/doc/libs/1_71_0/libs/python. 2019.



模拟集成神经系统(Analog Integrated Neural Systems)

硅脑(Brains in Silicon)

神经形态自适应可塑性可扩展电子系统(Systems of Neuromorphic Adaptive Plastic Scalable Electronics, SyNAPSE

IBM主导的SyNAPSE项目，在超级计算机上进行大脑皮层仿真的基础上，为了突破规模瓶颈，也开发了神经形态芯片TrueNorth。神经元采用简单的LIF模型，



https://www.yangfenzi.com/hangye/69039.html

https://www.yangfenzi.com/hangye/69592.html