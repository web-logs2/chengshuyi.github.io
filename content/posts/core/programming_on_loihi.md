---
title: "Programming Spiking Neural Networks on Intel’s Loihi"
date: 2020-05-29T17:03:40+08:00
description: ""
draft: true
tags: [类脑芯片]
categories: [类脑芯片]
---



主要展示了intuitive 基于python面向SNN的API、编译器和运行时库。作者主要描述了如何build、train和使用SNN去区分手写数字。



loihi[^1]manycore mesh包含128个类脑芯片、三个嵌入式IA核心（管理类脑芯片和控制脉冲在类脑芯片上的进出）和一个异步的片上网络（在类脑芯片之间传递信息）。

Programming this engine and the SNNs that use the microcodes requires toolchain features (such as dynamic rule swapping) that are not supported by current end-to-end software frameworks for neuromorphic hardware, such as PyNN,6 Nengo,7 and TrueNorth Corelets.8

![image-20200529171937233](C:\Users\dell\AppData\Roaming\Typora\typora-user-images\image-20200529171937233.png)

利用python提供复杂的SNN拓扑；编译器和runtime为了在loihi运行SNN

三个后端：一个软件的模拟器、一个FPGA仿真器和loihi

展示单层的SNN足以识别和区分手写数字，MNIST数据库。详细描述：从输入的图像的编码成脉冲、监督学习规则，到实现API和提取并解析结果



愿景是希望能够降低在loihi上面编程的强度。



### 编程模型

neuronal *compartments* and synaptic connections (or *synapses*, for short) as a means of defining SNN topology; *synaptic traces* and a *neuron* 

*model* to describe SNN dynamics; and synaptic *learning rules*. 

#### SNN 拓扑

有权重的有向图G(V,E)，顶点表示compartments，有权重的边E表示突触

神经元表示一系列的compartments被组织成树状拓扑结构（类似于树突的树形结构），在根部的compartment是唯一一个可以释放脉冲的

#### SNN dynamics



[^1]:M. Davies et al., “Loihi: a Neuromorphic Manycore Processor with On-Chip Learning,” *IEEE Micro*, vol. 38, no. 1, 2018.