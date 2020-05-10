---
title: "Lab5 - 实验笔记"
date: 2020-04-25T13:34:18+08:00
description: ""
draft: true
tags: [mit6828 OS实验]
categories: [mit6828 OS实验]
---

> **Exercise 1.** `i386_init` identifies the file system environment by passing the type `ENV_TYPE_FS` to your environment creation function, `env_create`. Modify `env_create` in `env.c`, so that it gives the file system environment I/O privilege, but never gives that privilege to any other environment.

解析：jos采用的文件系统ENV的类型包含两个，分别是`ENV_TYPE_USER`和`ENV_TYPE_FS`。

The x86 processor uses the IOPL bits in the EFLAGS register to determine whether protected-mode code is allowed to perform special device I/O instructions such as the IN and OUT instructions. Since all of the IDE disk registers we need to access are located in the x86's I/O space rather than being memory-mapped, giving "I/O privilege" to the file system environment is the only thing we need to do in order to allow the file system to access these registers. In effect, the IOPL bits in the EFLAGS register provides the kernel with a simple "all-or-nothing" method of controlling whether user-mode code can access I/O space. In our case, we want the file system environment to be able to access I/O space, but we do not want any other environments to be able to access I/O space at all.

```c
void
env_create(uint8_t *binary, enum EnvType type)
{
    // lab3
	struct Env *env;
	env_alloc(&env,0);
	env->env_type = type;
	load_icode(env,binary);
    // lab5
	if(type == ENV_TYPE_FS){
		env->env_type = ENV_TYPE_FS;
		env->env_tf.tf_eflags |= FL_IOPL_MASK;
	}
}
```

> **Question**
>
> 1. Do you have to do anything else to ensure that this I/O privilege setting is saved and restored properly when you subsequently switch from one environment to another? Why?

不需要，



> *Challenge!* Implement interrupt-driven IDE disk access, with or without DMA. You can decide whether to move the device driver into the kernel, keep it in user space along with the file system, or even (if you really want to get into the micro-kernel spirit) move it into a separate environment of its own.


>**Exercise 2.** Implement the `bc_pgfault` and `flush_block` functions in `fs/bc.c`. `bc_pgfault` is a page fault handler, just like the one your wrote in the previous lab for copy-on-write fork, except that its job is to load pages in from the disk in response to a page fault. When writing this, keep in mind that (1) `addr` may not be aligned to a block boundary and (2) `ide_read` operates in sectors, not blocks.
>
>The `flush_block` function should write a block out to disk *if necessary*. `flush_block` shouldn't do anything if the block isn't even in the block cache (that is, the page isn't mapped) or if it's not dirty. We will use the VM hardware to keep track of whether a disk block has been modified since it was last read from or written to disk. To see whether a block needs writing, we can just look to see if the `PTE_D` "dirty" bit is set in the `uvpt` entry. (The `PTE_D` bit is set by the processor in response to a write to that page; see 5.2.4.3 in [chapter 5](http://pdos.csail.mit.edu/6.828/2011/readings/i386/s05_02.htm) of the 386 reference manual.) After writing the block to disk, `flush_block` should clear the `PTE_D` bit using `sys_page_map`.Use make grade to test your code. Your code should pass "check_bc", "check_super", and "check_bitmap".

采用微内核的形式，文件系统驱动实现在用户态，读/写文件通过微内核提供的系统调用来实现。



```c
// static void bc_pgfault(struct UTrapframe *utf)
	// 地址对齐
	addr = ROUNDDOWN(addr,BLKSIZE);
	if((r = sys_page_alloc(0,(void *)addr,PTE_SYSCALL))<0)
		panic("in bc_pgfault, sys_page_alloc: %e",r);
	// 读取磁盘数据到内存中
	if((r = ide_read(((uintptr_t)addr-DISKMAP)/BLKSIZE*BLKSECTS,addr,BLKSECTS))<0)
		panic("in bc_pgfault, ide_read: %e",r);
```

```c
// void flush_block(void *addr)	
	int r;
	addr = ROUNDDOWN(addr,BLKSIZE);
	// 该地址对应的block存在内存中，并且有修改过（没有修改的话就不用flush）。
	if(va_is_mapped(addr) && va_is_dirty(addr)){
		if((r = ide_write(blockno*BLKSECTS,addr,BLKSECTS))<0)
			panic("in flush_block, ide_write %e",r);
        // 清除dirty位
		if ((r = sys_page_map(0, addr, 0, addr, uvpt[PGNUM(addr)] & PTE_SYSCALL)) < 0)
			panic("in flush_block, sys_page_map: %e", r);
	}
```

> *Challenge!* The block cache has no eviction policy. Once a block gets faulted in to it, it never gets removed and will remain in memory forevermore. Add eviction to the buffer cache. Using the `PTE_A` "accessed" bits in the page tables, which the hardware sets on any access to a page, you can track approximate usage of disk blocks without the need to modify every place in the code that accesses the disk map region. Be careful with dirty blocks.

> **Exercise 3.** Use `free_block` as a model to implement `alloc_block` in `fs/fs.c`, which should find a free disk block in the bitmap, mark it used, and return the number of that block. When you allocate a block, you should immediately flush the changed bitmap block to disk with `flush_block`, to help file system consistency.
>
> Use make grade to test your code. Your code should now pass "alloc_block".

从硬盘空闲块中分配一块，返回块号。

```c
//int alloc_block(void)
	uint32_t i;
	for(i=0;i<super->s_nblocks;i++){
		if(block_is_free(i)){
			bitmap[i/32] &= ~(1<<(i%32));
            // 将bitmap数据flush到硬盘，防止断电等意外发生，造成数据的丢失
			flush_block((void *)((2+i/BLKBITSIZE)*BLKSIZE+DISKMAP));
			return i;
		}
	}
```

> **Exercise 4.** Implement `file_block_walk` and `file_get_block`. `file_block_walk` maps from a block offset within a file to the pointer for that block in the `struct File` or the indirect block, very much like what `pgdir_walk` did for page tables. `file_get_block` goes one step further and maps to the actual disk block, allocating a new one if necessary.
>
> Use make grade to test your code. Your code should pass "file_open", "file_get_block", and "file_flush/file_truncated/file rewrite", and "testfile".

jos文件系统采用类似于inode的方式，每个文件数据对应的物理存储块编号采用直接方式和间接方式。直接索引就是将该存储块编号存放在inode的数组中，间接索引是将该存储块编号存放在block中，

逻辑块号和物理块号。文件通过逻辑块号

```c
//static int file_block_walk(struct File *f, uint32_t filebno, uint32_t **ppdiskbno, bool alloc)
	uintptr_t *addr;
	// 逻辑块号小于NDIRECT
	if(filebno < NDIRECT){
		*ppdiskbno = &f->f_direct[filebno];
		return 0;
	}else{
        // 超过文件最大限制
		if(filebno >= NDIRECT + NINDIRECT)
			return -E_INVAL;
        // 间接存储的block还未分配
		if(f->f_indirect== 0 || block_is_free(f->f_indirect)){
			if(alloc){
                // 分配一个block用于存储物理存储块编号
				f->f_indirect = alloc_block();
				if(f->f_indirect < 0){
					f->f_indirect = 0;
					return -E_NO_DISK;
				}
			}else{
				return -E_NOT_FOUND;
			}
		}
	}
	// 获取物理存储块编号
	addr = (uintptr_t *)(f->f_indirect*BLKSIZE+DISKMAP);
	*ppdiskbno = &addr[filebno-NDIRECT];
	return 0;
```

```c
// int file_get_block(struct File *f, uint32_t filebno, char **blk)
	// LAB 5: Your code here.
	int r;
	uint32_t *ppdiskbno;
	if((r = file_block_walk(f,filebno,&ppdiskbno,0))<0){
		return r;
	}
	// 物理块不存在，需要分配一个物理块给当前文件
	if(*ppdiskbno == 0){
		*ppdiskbno = alloc_block();
		if(*ppdiskbno < 0){
			return -E_NO_DISK;
		}
	}
	// 返回该物理块映射的虚拟地址
	*blk = (char *)diskaddr(*ppdiskbno);
	return 0;
```

> *Challenge!* The file system is likely to be corrupted if it gets interrupted in the middle of an operation (for example, by a crash or a reboot). Implement soft updates or journalling to make the file system crash-resilient and demonstrate some situation where the old file system would get corrupted, but yours doesn't.

> **Exercise 5.** Implement `serve_read` in `fs/serv.c`.
>
> `serve_read`'s heavy lifting will be done by the already-implemented `file_read` in `fs/fs.c` (which, in turn, is just a bunch of calls to `file_get_block`). `serve_read` just has to provide the RPC interface for file reading. Look at the comments and code in `serve_set_size` to get a general idea of how the server functions should be structured.
>
> Use make grade to test your code. Your code should pass "serve_open/file_stat/file_close" and "file_read" for a score of 70/150.



```c
//int serve_read(envid_t envid, union Fsipc *ipc)
	int r;
	struct OpenFile *o;
	// 检查该进程的打开文件列表是否存在该文件
	if((r=openfile_lookup(envid,req->req_fileid,&o))<0){
		return r;
	}
	// 读取文件数据（通过bc_pgfault的方式实现）
	r = file_read(o->o_file,ret->ret_buf,req->req_n,o->o_fd->fd_offset);
	// 修改seek位置
	if(r>0) o->o_fd->fd_offset += r;
	return r;
```

> **Exercise 6.** Implement `serve_write` in `fs/serv.c` and `devfile_write` in `lib/file.c`.
>
> Use make grade to test your code. Your code should pass "file_write", "file_read after file_write", "open", and "large file" for a score of 90/150.

```c
// int serve_write(envid_t envid, struct Fsreq_write *req)
	int r;
	struct OpenFile *o;
	// 检查该进程的打开文件列表是否存在该文件
	if((r=openfile_lookup(envid,req->req_fileid,&o))<0){
		return r;
	}
	r = file_write(o->o_file,req->req_buf,req->req_n > PGSIZE? PGSIZE:req->req_n,o->o_fd->fd_offset);
	if(r > 0) o->o_fd->fd_offset += r;
	return r;
```

```c
// static ssize_t devfile_read(struct Fd *fd, void *buf, size_t n)	
	int r;
	if ( n > sizeof (fsipcbuf.write.req_buf)) 
		n = sizeof (fsipcbuf.write.req_buf);
	memmove(fsipcbuf.write.req_buf, buf, n);
	fsipcbuf.write.req_fileid = fd->fd_file.id;
	fsipcbuf.write.req_n = n;
	return fsipc(FSREQ_WRITE, NULL);
```

> **Exercise 7.** `spawn` relies on the new syscall `sys_env_set_trapframe` to initialize the state of the newly created environment. Implement `sys_env_set_trapframe` in `kern/syscall.c` (don't forget to dispatch the new system call in `syscall()`).
>
> Test your code by running the `user/spawnhello` program from `kern/init.c`, which will attempt to spawn `/hello` from the file system.
>
> Use make grade to test your code.

```c
// static int sys_env_set_trapframe(envid_t envid, struct Trapframe *tf)	
	struct Env *e;
	if(envid2env(envid,&e,1) < 0){
		return -E_BAD_ENV;
	}
	user_mem_assert(e, tf, sizeof(struct Trapframe), PTE_U);
	e->env_tf = *tf;
	e->env_tf.tf_eflags |= FL_IF;
	e->env_tf.tf_eflags &= (~FL_IOPL_MASK);
	return 0;
```

> **Exercise 8.** Change `duppage` in `lib/fork.c` to follow the new convention. If the page table entry has the `PTE_SHARE` bit set, just copy the mapping directly. (You should use `PTE_SYSCALL`, not `0xfff`, to mask out the relevant bits from the page table entry. `0xfff` picks up the accessed and dirty bits as well.)
>
> Likewise, implement `copy_shared_pages` in `lib/spawn.c`. It should loop through all page table entries in the current process (just like `fork` did), copying any page mappings that have the `PTE_SHARE` bit set into the child process.

spawn的话可以从用户态加载应用程序去运行

