---
title: "Lab2 - 实验笔记"
date: 2020-04-25T13:34:11+08:00
description: ""
draft: false
tags: [mit6828 OS实验]
categories: [mit6828 OS实验]
---

[lab2实验代码](https://github.com/chengshuyi/jos-lab/commit/ca384c1373d3a34cfa82435fdd366965a3b4c071)

lab2的实验主要涉及到物理内存和虚拟内存的管理，其中JOS内核按页对物理内存进行管理，所以其分配、释放的最小内存大小是一个物理页。相应的需要实现三个物理页相关函数：

* `void page_init(void)`
* `struct PageInfo *page_alloc(int alloc_flags)`
* `void page_free(struct PageInfo *pp)`

JOS内核的虚拟地址空间大小是256MB，x86中包含分段机制和分页机制，JOS分段机制比较固定，占用整个地址空间大小。因此，整个实验比较关注于基于mmu的分页管理机制。针对分页机制需要实现四个函数：

* `pte_t *pgdir_walk(pde_t *pgdir, const void *va, int create)`
* `void page_remove(pde_t *pgdir, void *va)`
* `struct PageInfo *page_lookup(pde_t *pgdir, void *va, pte_t **pte_store)`
* `int page_insert(pde_t *pgdir, struct PageInfo *pp, void *va, int perm)`

本文不会涉及到具体代码，简单记录了比较关键的部分。

### 物理页管理

JOS的物理内存粒度是页，所有的空闲页通过链表进行管理，下面是内核物理内存管理的流程，特别需要注意到在没有初始化内存之前，内核如何为数据结构分配内存。

1. 通过I/O端口获取物理内存的分布情况，分为两部分，base memory占用640KB，extended memory占用130432KB。可以知道base memory指的是[lab1]()讲的low memory；

   > Physical memory: 131072K available, base = 640K, extended = 130432K

2. 实现`boot_alloc`函数，该函数主要完成在没有建立页管理机制之前的内存分配工作。其工作流程是：

   a. 根据链接脚本的end参数获取内核代码所占内存的终止地址，并将其按页对齐；

   b. 为调用者分配内存，更新终止位置；

3. 调用`boot_alloc`函数为内核页目录表`kern_pgdir`分配内存，大小是PGSIZE，二级页表在做映射的时候才进行分配；

4. 建立自映射，这个在lab4还会遇到；

5. 调用`boot_alloc`函数为管理页数据结构`struct PageInfo`分配内存，其内存大小是`sizeof(struct PageInfo)*npages`；

6. `page_init`函数初始化`struct PageInfo`，主要是将空闲页挂在空闲页链表`page_free_list`上，此时空闲页主要有两部分构成：1. low memory部分； 2. extended memory剩余部分；

7. `page_init`函数从空闲页链表上取出一页；

8. `page_free`函数将该页挂在空闲链表上；

### 虚拟内存

在x86架构中，有三种地址术语，分别是：虚拟地址、线性地址和物理地址。虚拟地址通过分段机制转换得到线性地址，线性地址通过分页机制转换可以得到物理地址。

```c

           Selector  +--------------+         +-----------+
          ---------->|              |         |           |
                     | Segmentation |         |  Paging   |
Software             |              |-------->|           |---------->  RAM
            Offset   |  Mechanism   |         | Mechanism |
          ---------->|              |         |           |
                     +--------------+         +-----------+
            Virtual                   Linear                Physical
```

关于分段机制，现在操作系统用的很少，大部分将段基地址设置成`0`，段的大小是`0xffffffff`，所以虚拟地址等于线性地址。

> 分段机制目前用的比较多的是它的特权等级，jos中分为内核段和用户段；

在lab1中只映射了4MB的物理内存，在qemu中运行`info pg`可以得到：

```c
VPN range     Entry         Flags        Physical page
[00000-003ff]  PDE[000]     ----A----P
  [00000-000b7]  PTE[000-0b7] --------WP 00000-000b7
  ...
  [00114-003ff]  PTE[114-3ff] --------WP 00114-003ff
[f0000-f03ff]  PDE[3c0]     ----A---WP
  [f0000-f00b7]  PTE[000-0b7] --------WP 00000-000b7
  ...
  [f0114-f03ff]  PTE[114-3ff] --------WP 00114-003ff
```

从上面可以看到，物理内存[0,4MB)分别映射到虚拟内存[0,4MB)和[0xf0000000,0xf03ff000)。

```c
/*
 * Virtual memory map:                                Permissions
 *                                                    kernel/user
 *
 *    4 Gig -------->  +------------------------------+
 *                     |                              | RW/--
 *                     ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
 *                     :              .               :
 *                     :              .               :
 *                     :              .               :
 *                     |~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~| RW/--
 *                     |                              | RW/--
 *                     |   Remapped Physical Memory   | RW/--
 *                     |                              | RW/--
 *    KERNBASE, ---->  +------------------------------+ 0xf0000000      --+
 *    KSTACKTOP        |     CPU0's Kernel Stack      | RW/--  KSTKSIZE   |
 *                     | - - - - - - - - - - - - - - -|                   |
 *                     |      Invalid Memory (*)      | --/--  KSTKGAP    |
 *                     +------------------------------+                   |
 *                     |     CPU1's Kernel Stack      | RW/--  KSTKSIZE   |
 *                     | - - - - - - - - - - - - - - -|                 PTSIZE
 *                     |      Invalid Memory (*)      | --/--  KSTKGAP    |
 *                     +------------------------------+                   |
 *                     :              .               :                   |
 *                     :              .               :                   |
 *    MMIOLIM ------>  +------------------------------+ 0xefc00000      --+
 *                     |       Memory-mapped I/O      | RW/--  PTSIZE
 * ULIM, MMIOBASE -->  +------------------------------+ 0xef800000
 *                     |  Cur. Page Table (User R-)   | R-/R-  PTSIZE
 *    UVPT      ---->  +------------------------------+ 0xef400000
 *                     |          RO PAGES            | R-/R-  PTSIZE
 *    UPAGES    ---->  +------------------------------+ 0xef000000
 *                     |           RO ENVS            | R-/R-  PTSIZE
 * UTOP,UENVS ------>  +------------------------------+ 0xeec00000
 * UXSTACKTOP -/       |     User Exception Stack     | RW/RW  PGSIZE
 *                     +------------------------------+ 0xeebff000
 *                     |       Empty Memory (*)       | --/--  PGSIZE
 *    USTACKTOP  --->  +------------------------------+ 0xeebfe000
 *                     |      Normal User Stack       | RW/RW  PGSIZE
 *                     +------------------------------+ 0xeebfd000
 *                     |                              |
 *                     |                              |
 *                     ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
 *                     .                              .
 *                     .                              .
 *                     .                              .
 *                     |~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~|
 *                     |     Program Data & Heap      |
 *    UTEXT -------->  +------------------------------+ 0x00800000
 *    PFTEMP ------->  |       Empty Memory (*)       |        PTSIZE
 *                     |                              |
 *    UTEMP -------->  +------------------------------+ 0x00400000      --+
 *                     |       Empty Memory (*)       |                   |
 *                     | - - - - - - - - - - - - - - -|                   |
 *                     |  User STAB Data (optional)   |                 PTSIZE
 *    USTABDATA ---->  +------------------------------+ 0x00200000        |
 *                     |       Empty Memory (*)       |                   |
 *    0 ------------>  +------------------------------+                 --+
```

上面是虚拟内存地址空间分布的情况，下面是建立映射的大致过程：

1. 前面已经为页目录表`kern_pgdir`分配了内存，所以接下来的建立地址映射的过程就是在`kern_pgdir`插入页表项，以及分配页表；
2. 将前面分配的`sizeof(struct PageInfo) * npage`映射到UPAGES；
3. 将内核栈映射到KSTACKTOP；
4. 将整个物理内存映射内核虚拟地址空间；
5. 将`kern_pgdir`加载到cr3寄存器；

下面是几个函数的实现功能，描述的比较简单，但实际上有很多的细节需要去考虑：

1. `pgdir_walk`函数查找虚拟地址所对应的页表项，返回该页表项的指针，也就是`&kern_pgdir[PDX(va)][PTX(va)]`；

2. `page_lookup`函数查找虚拟地址对应的物理页，也就是`pa2page(kern_pgdir[PDX(va)][PTX(va)])`；

3. `page_remove`函数完成两个功能：

   a. 取消虚拟地址映射；

   b. 回收物理页。

   c. 取消映射原理是`kern_pgdir[PDX(va)][PTX(va)] = 0`，回收物理页原理将该页插入到空闲页链表即可。

4. `page_insert`函数对该页建立映射，也就是`kern_pgdir[PDX(va)][PTX(va)]=PGNUM(va)`；

