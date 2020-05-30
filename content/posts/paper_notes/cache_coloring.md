---
title: "Cache_coloring"
date: 2020-05-30T19:41:43+08:00
description: ""
draft: true
tags: []
categories: []
---

摘要：软件使用不同的优化策略提高cache的性能，比如：software prefetching、data rescheduling and code reodrdering。
本文属于code reordering，

降低指令cache的miss率，大概是40%

简介

1. cache miss的原因有：cold start, 容量和冲突
2. code reordering能够降低程序的运行时间，大概有15%
3. 本文的主要研究特点在于，使用cache size, cache line size和程序大小去完成color mapping


page coloring技术应该不是指代保证虚实地址一致的技术。 > 为了保证虚实地址一致，一个简单办法是保证同一物理页对应的两个虚页地址距离大于cache size（见《See MIPS RUN》4.14.2 Cache Aliases）。而page coloring是一种优化策略（这 意味着它不是必须的），而消除aliases却不是优化问题，而是一个必须解决的问题。关于pag e coloring： http://www.freebsd.org/doc/en_US.ISO8859-1/articles/vm-design/page-coloring-optimizations.html

为了避免Cache替换，不同的数据结构的地址对应的cache line索引最好不要相同，否则冲突的概率增大。

而使用slab的数据结构都是分配和释放频繁的小的数据结构，而且数目很多，比如dentry，如果没有color，他们在内存中相对于页的偏移量很可能相同，则其cache line索引也相同，对于x86这种2way 的cache结构，即使cache size很大，也一样使用率低下。color则将不同slab中的同样的数据结构的地址进行一个偏移，因此这些数据结构的cache line索引就错开了。从而能更好的利用cache 
————————————————
版权声明：本文为CSDN博主「maray」的原创文章，遵循CC 4.0 BY-SA版权协议，转载请附上原文出处链接及本声明。
原文链接：https://blog.csdn.net/maray/article/details/3599845

.intel core i7 cache hierarchy

core内部：L1 cache（区分Data和Instruction）和L2 unified cache
core共享：L3 unified cache
参数：L1（32kB，8-way，4 cycles）、L2（256KB，8-way，10 cycles）、L3（8MB，16-way，40-75 cycles）
block size：64B
————————————————
版权声明：本文为CSDN博主「Mr0cheng」的原创文章，遵循CC 4.0 BY-SA版权协议，转载请附上原文出处链接及本声明。
原文链接：https://blog.csdn.net/Mr0cheng/article/details/104770517/