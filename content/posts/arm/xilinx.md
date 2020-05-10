---
title: "Xilinx"
date: 2020-04-29T17:34:47+08:00
description: ""
draft: true
tags: []
categories: []
---





jtag不识别问题



首先，想到的两个文件就是 PL 部分需要的 bit 文件，以及 PS 需要的 elf 文件。但是仅有这两个文件不够的。

我们还需要一段代码把 bit 文件以及 elf 文件安置好。这段代码就是大名鼎鼎的 FSBL.elf。

BOOT.bin = FSBL.elf+该工程.bit+该工程.elf

PS（processing system），就是与FPGA无关的ARM的SOC的部分；

PL（可编程逻辑），也就是FPGA部分；

**SD 固化**：将镜像文件拷贝到 SD 卡，设置拨码开关，使系统从 SD 模式启动。那么每次断电重启后，系统都会

从 SD 启动。

**QSPI FLASH** **固化**：设置拨码开关，将镜像文件烧写进 FLASH，使系统从 QSPI-FLASH 模式启动。那么每次

断电重启后，系统都会从 FLASH 启动。

**启动模式** 

**开关状态**

SD 启动/JTAG 调试模式 

开关 1-OFF，开关 2-OFF

QSPI FLASH 启动/JTAG 调试模式 

开关 1-ON ，开关 2-OFF

### BOOT.bin制作过程

step1：创建vivado工程，添加QSPI flash接口；添加SDIO接口；添加串口；

step2：导入SDK;

step3：创建应用工程；

step4：创建FSBL工程；

step5：右键应用工程，选择 Creat Boot Image

### SD-TF卡启动

将生成的 BOOT.bin（名字必须是这个，不然不识别） 文件，复制到 SD 卡，再将 SD 卡插到开发板，最后打开电源。则开机后系统从 SD 卡启动，程序掉电不消失。

### QSPI-FLASH 下载方法如下

Step1: 新建环境变量：计算机->属性->高级系统设置->高级->环境变量->新建系统变量

​	变量名：XIL_CSE_ZYNQ_UBOOT_QSPI_FREQ_HZ

​	变量值：10000000

Step2: 生成加载 QSPI FLASH 的 fsbl 文件：新建一个新的 FSBL 文件，命名为 zynq_fsbl。File->New->Application Project，输入 zynq_fsbl，点击 Next。选择Zynq FSBL，单击 Finish。

Step3: 打开 zynq_fsbl 的 main.c 文件，在此处增加`BootModeRegister = JTAG_MODE;`保存并编译

![](https://gitee.com/chengshuyi/scripts/raw/master/img/20200429174920.png)

Step4: 模式开关切换到 QSPI 启动模式（1-ON ,2-OFF），开发板通电。选择 Xilinx Tools > Program Flash 或单击

Program Flash Memory。

Step6:下载完成后断电，重新打开电源，就能看到从 QSPI FLASH 加载。





https://blog.csdn.net/maddisonn/article/details/88062796





工作记录：

1. debug应该使用右键项目debug as，不要使用快捷栏得按钮；
2. sd path的编号是逻辑编号，一般0是sd卡，1是emmc，但是我只是用了emmc，所以emmc的path变成了0；
3. f_mkfs是初始化整个emmc，重新写入fat表，所以调用一次后就可以注释掉了
4. 



修改Lwip v1.41置为nosysnotimer false