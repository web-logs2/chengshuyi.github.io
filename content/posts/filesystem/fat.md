---
title: "Fat32文件系统详解"
date: 2020-05-14T20:06:37+08:00
description: ""
draft: false
tags: [文件系统]
categories: [文件系统]
---



本文主要介绍了fat32文件系统，包括文件分配表和目录项等重要内容。



### fat32磁盘空间的划分

首先第一个扇区（512字节）作为引导扇区，其中引导扇区记录了当前硬盘的分区情况，以及每个分区所包含的信息：文件系统类型、分区开始的扇区和占用扇区的个数；

然后根据对应的分区可以找到该分区的第一个扇区，该扇区称为`volume ID`。`volume ID`包含了一些关于当前文件系统的重要信息，以fat32为例：

* 每个扇区占多少字节（总是512字节）；
* 每个簇有多少个扇区（2的n次方）
* 保留扇区的个数；
* 文件分配表的个数（总是2，有一个是备份）；
* 文件分配表占用的扇区数（根据分区大小计算即可）；
* 根目录所在的簇（一般是2）；

接着是保留扇区空间；

再接着是文件分配表区（显示链接法）：

* fat32说明其文件分配表的表项是32位的；
* 每一个表项的位置代表着一个簇号，该表项的值表示下一个簇号；

最后是数据区，不仅包含了文件数据还包含了文件系统数据。

### 引导区

硬盘的第一个扇区称为引导区，也叫主引导扇区（MBR）。Boot Code也就是我们常说的bootloader代码。partition n表示分区，可以看到最多支持四个分区。

![](https://gitee.com/chengshuyi/scripts/raw/master/img/20200514222207.png)

这里我们主要看下分区描述符所占的16字节包含哪些信息？格式如下图所示：

![](https://gitee.com/chengshuyi/scripts/raw/master/img/20200602135143.png)

重要的信息有三个：

1. Type code：表明当前文件系统的类型；
2. LBA Begin：当前分区起始扇区号；
3. Number of Sectors：当前分区占有多少扇区；

### volume ID

分区的第一个扇区称为`volume ID`。`volume ID`包含了一些关于当前文件系统的重要信息：

* 每个扇区占多少字节（总是512字节）；
* 每个簇有多少个扇区（2的n次方）
* 保留扇区的个数；
* 文件分配表的个数（总是2，有一个是备份）；
* 文件分配表占用的扇区数（根据分区大小计算即可）；
* 根目录所在的簇（一般是2）；

![](https://gitee.com/chengshuyi/scripts/raw/master/img/20200602135838.png)

### 保留扇区

![](https://gitee.com/chengshuyi/scripts/raw/master/img/20200602161814.png)

### 文件分配表区





### 数据区

![](https://gitee.com/chengshuyi/scripts/raw/master/img/20200602162221.png)

### 专业名词缩写

LBA:logical block addressing

MBR:master Boot Record

CHS:Cylinder Head Sector



### 参考文献

[1]. Understanding FAT32 Filesystems. https://www.pjrc.com/tech/8051/ide/fat32.html.