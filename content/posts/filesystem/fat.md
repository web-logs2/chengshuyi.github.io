---
title: "Fat"
date: 2020-05-14T20:06:37+08:00
description: ""
draft: false
tags: []
categories: []
---



以8G的emmc为例，采用sector为512B、cluster





本文主要介绍了fat32文件系统：

1. fat32将硬盘

the 32 part of its name comes from the fact that *FAT32* uses 32 bits of data for identifying data clusters on the storage device.



every [cluster](http://www.data-recovery-app.com/datarecovery/cluster.html) is fixed to 4KB

sectors that are 512 bytes.



The first sector of the drive is called the Master Boot Record (MBR).

![](https://gitee.com/chengshuyi/scripts/raw/master/img/20200514222207.png)



![](https://gitee.com/chengshuyi/scripts/raw/master/img/20200514222305.png)







### 物理布局







```c
typedef struct {
    // 3
	BYTE	fs_type;		/* FAT sub-type (0:Not mounted) */
	BYTE	drv;			/* Physical drive number */
	BYTE	csize;			/* Sectors per cluster (1,2,4...128) */
	BYTE	n_fats;			/* Number of FAT copies (1 or 2) */
	BYTE	wflag;			/* win[] flag (b0:dirty) */
	BYTE	fsi_flag;		/* FSINFO flags (b7:disabled, b0:dirty) */
	WORD	id;				/* File system mount ID */
	WORD	n_rootdir;		/* Number of root directory entries (FAT12/16) */
#if _MAX_SS != _MIN_SS
	WORD	ssize;			/* Bytes per sector (512, 1024, 2048 or 4096) */
#endif
#if _FS_REENTRANT
	_SYNC_t	sobj;			/* Identifier of sync object */
#endif
#if !_FS_READONLY
	DWORD	last_clust;		/* Last allocated cluster */
	DWORD	free_clust;		/* Number of free clusters */
#endif
#if _FS_RPATH
	DWORD	cdir;			/* Current directory start cluster (0:root) */
#endif
	DWORD	n_fatent;		/* Number of FAT entries, = number of clusters + 2 */
	DWORD	fsize;			/* Sectors per FAT */
	DWORD	volbase;		/* Volume start sector */
	DWORD	fatbase;		/* FAT start sector */
	DWORD	dirbase;		/* Root directory start sector (FAT32:Cluster#) */
	DWORD	database;		/* Data start sector */
	DWORD	winsect;		/* Current sector appearing in the win[] */
#ifdef __ICCARM__
#pragma data_alignment = 32
	BYTE	win[_MAX_SS];
#pragma data_alignment = 4
#else
	BYTE	win[_MAX_SS] __attribute__ ((aligned(32)));	/* Disk access window for Directory, FAT (and Data on tiny cfg) */
#endif
} FATFS;
```



f_mount

​	check_fs：加载第一个sector

```c
typedef struct {
	FATFS*	fs;				/* Pointer to the related file system object (**do not change order**) */
	WORD	id;				/* Owner file system mount ID (**do not change order**) */
	BYTE	flag;			/* Status flags */
	BYTE	err;			/* Abort flag (error code) */
	DWORD	fptr;			/* File read/write pointer (Zeroed on file open) 当前文件的指针，按字节计算*/
	DWORD	fsize;			/* File size 文件的实际大小*/ 
	DWORD	sclust;			/* File start cluster (0:no cluster chain, always 0 when fsize is 0) */
	DWORD	clust;			/* Current cluster of fpter (not valid when fprt is 0) */
	DWORD	dsect;			/* Sector number appearing in buf[] (0:invalid) */
#if !_FS_READONLY
	DWORD	dir_sect;		/* Sector number containing the directory entry */
	BYTE*	dir_ptr;		/* Pointer to the directory entry in the win[] */
#endif
#if _USE_FASTSEEK
	DWORD*	cltbl;			/* Pointer to the cluster link map table (Nulled on file open) */
#endif
#if _FS_LOCK
	UINT	lockid;			/* File lock ID origin from 1 (index of file semaphore table Files[]) */
#endif
#if !_FS_TINY
#ifdef __ICCARM__
#pragma data_alignment = 32
	BYTE	buf[_MAX_SS];	/* File data read/write buffer */
#pragma data_alignment = 4
#else
	BYTE	buf[_MAX_SS] __attribute__ ((aligned(32)));	/* File data read/write buffer */
#endif
#endif
} FIL;
```



```c
typedef struct {
	FATFS*	fs;				/* Pointer to the owner file system object (**do not change order**) */
	WORD	id;				/* Owner file system mount ID (**do not change order**) */
	WORD	index;			/* Current read/write index number */
	DWORD	sclust;			/* Table start cluster (0:Root dir) */
	DWORD	clust;			/* Current cluster */
	DWORD	sect;			/* Current sector */
	BYTE*	dir;			/* Pointer to the current SFN entry in the win[] */
	BYTE*	fn;				/* Pointer to the SFN (in/out) {file[8],ext[3],status[1]} */
#if _FS_LOCK
	UINT	lockid;			/* File lock ID (index of file semaphore table Files[]) */
#endif
#if _USE_LFN
	WCHAR*	lfn;			/* Pointer to the LFN working buffer */
	WORD	lfn_idx;		/* Last matched LFN index number (0xFFFF:No LFN) */
#endif
} DIR;
```





![](https://gitee.com/chengshuyi/scripts/raw/master/img/20200515101450.png)

![](https://gitee.com/chengshuyi/scripts/raw/master/img/20200515101534.png)

![](https://gitee.com/chengshuyi/scripts/raw/master/img/20200515101741.png)

![](https://gitee.com/chengshuyi/scripts/raw/master/img/20200515101833.png)





op is 5
tick time is 35b60
op is 1
packet handler
packet handler
packet handler
packet handler
mode handler
op is 2
len is 197
tick handle
packet handler
op is 2
len is 168
tick handle
packet handler