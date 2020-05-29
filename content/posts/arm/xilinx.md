---
title: "Xilinx"
date: 2020-04-29T17:34:47+08:00
description: ""
draft: true
tags: []
categories: []
---

### 2020年5月14日
1. 解决文件传输问题，读/写入文件出现FR_INVALID_OBJECT问题，原因是FIL file全局变量没有初始化；
2. 文件名长度不能超过8个字符

### 2020年5月18日

解决input文件传输问题：

1. 先移植tcp传输Input file，发现仍然传输不了，怀疑是config配置文件有问题；
2. 测试是否配置文件发错时是否会影响input file的传输；（用戴书画原来的文件测试发现会）
3. 定位到config发送时出现的问题；


jtag不识别问题

### 2020年5月20日

1. 解决上下文切换后，程序跑飞的问题（重新移植osek）

### 2020年5月27日

![img](https://images2018.cnblogs.com/blog/867021/201803/867021-20180322001733298-201433635.jpg)

1. 解决多次Input出现卡死的问题（配置文件有问题）

### 2020年5月28日

1. 删除row文件，直接从input文件解析
2. 解决多次（几十次）input出现卡死的问题（重新生成输入文件）
3. 修改提取文件名函数，和师弟保持一致
4. 解决文件传输问题，由于匹配前缀，导致文件传输失败；
5. 解决文件名字问题，主要是memcpy后，没有在字符串末尾加'\0'



首先，想到的两个文件就是 PL 部分需要的 bit 文件以及 PS 需要的 elf 文件。但是仅有这两个文件不够的。

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
4. 网络ping不通，修改为固定速率1000M
5. xilffs undefined reference to f_open  https://blog.csdn.net/weixin_44167319/article/details/104074045
