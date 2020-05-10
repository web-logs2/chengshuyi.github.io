---
title: "Ffs"
date: 2020-04-27T14:51:15+08:00
description: ""
draft: true
tags: []
categories: []
---



### 2. Old file system

disk drive

​	one or more partitions

​	one partitions one file system

​	one file system one superblock

​	one superblock:

​		the number of data blocks

​		a count of maximum number of files

​		a pointer to the free list

​	files:

​		directories

​			pointers to files

​	inode

​		time stamps(creation modification access)

​		arrary of indices of data blocks

​		indrect block

​		

### 3. new file system organization

disk drive

​	one or more cylinder groups

​	one or more consecutive cylinders

​	bookkeeping info	

​		superblock

​		inodes

​		bitmap