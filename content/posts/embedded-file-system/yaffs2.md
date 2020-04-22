---
title: "Yaffs2"
date: 2020-04-21T14:11:03+08:00
description: ""
draft: true
tags: []
categories: []
---





| field         | comment                             | size for 1kb chunks | size for 2kb chunks |
| ------------- | ----------------------------------- | ------------------- | ------------------- |
| blockState    | Block state. non-0xFF for bad block | 1 byte              | 1 byte              |
| chunkId       | 32-bit chunk Id                     | 4 bytes             | 4 bytes             |
| objectId      | 32-bit object Id                    | 4 bytes             | 4 bytes             |
| nBytes        | Number of data bytes in this chunk  | 2 bytes             | 2 bytes             |
| blockSequence | sequence number for this block      | 4 bytes             | 4 bytes             |
| tagsEcc       | ECC on tags area                    | 3 bytes             | 3 bytes             |
| ecc           | ECC, 3 bytes/256 bytes of data      | 12 bytes            | 24 bytes            |
| **Total**     |                                     | **30 bytes**        | **42 bytes**        |

YAFFS2 并不扫描所有闪存页的内容，而仅扫描页空闲区和对象头所在的页
就能建立整个文件系统的文件索引结构 Tnode 树，从而加快了文件系统的启动挂载速度。
当文件索引信息发生改变时，YAFFS2 只需在内存中修改 Tnode 树即可，而无需同步地写
入闪存，加快了数据读写速度，同时也减少了系统对闪存块不必要的擦除操作。此外，检
查点 checkpoint 的引入更进一步缩短了文件系统的启动时间。