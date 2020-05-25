---
title: "Lab5 - 实验笔记"
date: 2020-04-25T13:34:18+08:00
description: ""
draft: false
tags: [mit6828 OS实验]
categories: [mit6828 OS实验]
---

### 文件系统架构

在`init.c`通过`ENV_CREATE(fs_fs, ENV_TYPE_FS);`创建一个专用于文件系统的进程，该进程提供了文件系统的管理，包括读取文件、写入文件等等；

假设有其他的进程需要进行文件读取，则需要通过ipc（inter process call）的方式将读取文件的元信息传递文件系统进程；

以打开文件为例，具体代码流程如下：

```c
// 库提供的open函数
int open(const char *path, int mode)
{
	int r;
	struct Fd *fd;
	// fd_alloc利用之前的自映射快速找到用户进程空间未使用的struct Fd空间。（通过判断虚拟地址对应的表项是否存在）
	if ((r = fd_alloc(&fd)) < 0)
		return r;
	// ipc参数
	strcpy(fsipcbuf.open.req_path, path);
	fsipcbuf.open.req_omode = mode;
	// fsipc的信息传递，具体看下面
	if ((r = fsipc(FSREQ_OPEN, fd)) < 0) {
		fd_close(fd, 0);
		return r;
	}
	return fd2num(fd);
}
```

```c
// file system对jos提供的ipc机制的封装，主要是传递的参数格式化
// type是操作类型，比如FSREQ_OPEN
// dstva是struct Fd的虚拟地址
static int fsipc(unsigned type, void *dstva)
{
	static envid_t fsenv;
	if (fsenv == 0)
        // 利用子映射快速遍历所有的env，找到文件系统对应的env（ipc通信需要的参数）
		fsenv = ipc_find_env(ENV_TYPE_FS);
    // ipc
	ipc_send(fsenv, type, &fsipcbuf, PTE_P | PTE_W | PTE_U);
	return ipc_recv(NULL, dstva, NULL);
}
```



```c
      Regular env           FS env
   +---------------+   +---------------+
   |      read     |   |   file_read   |
   |   (lib/fd.c)  |   |   (fs/fs.c)   |
...|.......|.......|...|.......^.......|...............
   |       v       |   |       |       | RPC mechanism
   |  devfile_read |   |  serve_read   |
   |  (lib/file.c) |   |  (fs/serv.c)  |
   |       |       |   |       ^       |
   |       v       |   |       |       |
   |     fsipc     |   |     serve     |
   |  (lib/file.c) |   |  (fs/serv.c)  |
   |       |       |   |       ^       |
   |       v       |   |       |       |
   |   ipc_send    |   |   ipc_recv    |
   |       |       |   |       ^       |
   +-------|-------+   +-------|-------+
           |                   |
           +-------------------+
```

### spawn

spawn是创建进程的另外一种方式，它不同于fork。fork的话是创建和自身一样的运行环境，程序代码也是当前运行的程序代码。而spawn则是从文件系统加载一段新的程序代码去运行。spwan的具体流程如下：

1. 加载程序代码到内存中（从文件系统读取程序代码）；
2. 创建子进程；



### shell



```c
struct Pipe {
	off_t p_rpos;		// read position
	off_t p_wpos;		// write position
	uint8_t p_buf[PIPEBUFSIZ];	// data buffer
};

struct Dev devpipe =
{
	.dev_id =	'p',
	.dev_name =	"pipe",
	.dev_read =	devpipe_read,
	.dev_write =	devpipe_write,
	.dev_close =	devpipe_close,
	.dev_stat =	devpipe_stat,
};

// 分开的命令分别进行spawn出新的进程去处理
		case '|':	
			// 分配出两个fd,绑定的文件是devpipe
			if ((r = pipe(p)) < 0) {
				cprintf("pipe: %e", r);
				exit();
			}
			// 创建子进程
			if ((r = fork()) < 0) {
				cprintf("fork: %e", r);
				exit();
			}
			if (r == 0) {
                // 子进程
				if (p[0] != 0) {
                    // 0是stdin，将pipe的第一个fd变成子进程的stdin
					dup(p[0], 0);
                    // 释放pipe的fd
					close(p[0]);
				}
                // 释放pipe的fd
				close(p[1]);
				goto again;
			} else {
                // 当前进程
				pipe_child = r;
				if (p[1] != 1) {
                    // 将pipe的第二个fd变成当前进程的stdout
					dup(p[1], 1);
					close(p[1]);
				}
				close(p[0]);
				goto runit;
			}
```







> **Exercise 1.** `i386_init` identifies the file system environment by passing the type `ENV_TYPE_FS` to your environment creation function, `env_create`. Modify `env_create` in `env.c`, so that it gives the file system environment I/O privilege, but never gives that privilege to any other environment.



标记不同env的类型，文件系统的env类型是`ENV_TYPE_FS`。修改flags让用户模式的程序可以使用io特权指令。

<!-- The x86 processor uses the IOPL bits in the EFLAGS register **to determine whether protected-mode code is allowed to perform special device I/O instructions such as the IN and OUT instructions.** Since all of the IDE disk registers we need to access are located in the x86's I/O space rather than being memory-mapped, **giving "I/O privilege" to the file system environment is the only thing we need to do in order to allow the file system to access these registers**. In effect, the IOPL bits in the EFLAGS register provides the kernel with a simple "all-or-nothing" method of controlling whether user-mode code can access I/O space. In our case, we want the file system environment to be able to access I/O space, but we do not want any other environments to be able to access I/O space at all. -->

```c
void env_create(uint8_t *binary, enum EnvType type)
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


>**Exercise 2.** Implement the `bc_pgfault` and `flush_block` functions in `fs/bc.c`. `bc_pgfault` is a page fault handler, just like the one your wrote in the previous lab for copy-on-write fork, except that its job is to load pages in from the disk in response to a page fault. When writing this, keep in mind that (1) `addr` may not be aligned to a block boundary and (2) `ide_read` operates in sectors, not blocks.

硬盘的数据在内存中叫block cache，jos分配了固定的虚拟地址空间存放硬盘数据，当我们需要读写的时候，如果该数据在内存中不存在，则会触发异常，该异常由内核传递给用户程序处理，即`bc_pgfault`函数。

```c
// static void bc_pgfault(struct UTrapframe *utf)
	// 地址对齐
	addr = ROUNDDOWN(addr,BLKSIZE);
	// 分配一个page的物理内存
	if((r = sys_page_alloc(0,(void *)addr,PTE_SYSCALL))<0)
		panic("in bc_pgfault, sys_page_alloc: %e",r);
	// 读取磁盘数据到内存中
	if((r = ide_read(((uintptr_t)addr-DISKMAP)/BLKSIZE*BLKSECTS,addr,BLKSECTS))<0)
		panic("in bc_pgfault, ide_read: %e",r);
```

`flush_block`主要是维持内存和硬盘中的数据同步。

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

> **Exercise 3.** Use `free_block` as a model to implement `alloc_block` in `fs/fs.c`, which should find a free disk block in the bitmap, mark it used, and return the number of that block. When you allocate a block, you should immediately flush the changed bitmap block to disk with `flush_block`, to help file system consistency.

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

jos文件系统采用类似于inode的方式，每个文件数据对应的物理存储块编号采用直接方式和间接方式。直接索引就是将该存储块编号存放在inode的数组中，间接索引是将该存储块编号存放在block中。`file_block_walk`函数将文件的逻辑地址转换成对应的硬盘上的物理地址。

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
	// 获取间接索引的物理存储块（一般会触发上面的bc_pgfault）
	addr = (uintptr_t *)(f->f_indirect*BLKSIZE+DISKMAP);
	// 物理块号
	*ppdiskbno = &addr[filebno-NDIRECT];
	return 0;
```

```c
// int file_get_block(struct File *f, uint32_t filebno, char **blk)
	// LAB 5: Your code here.
	int r;
	uint32_t *ppdiskbno;
	// 找到逻辑块号对应的物理块号
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

> **Exercise 5.** Implement `serve_read` in `fs/serv.c`.
>
> `serve_read`'s heavy lifting will be done by the already-implemented `file_read` in `fs/fs.c` (which, in turn, is just a bunch of calls to `file_get_block`). `serve_read` just has to provide the RPC interface for file reading. Look at the comments and code in `serve_set_size` to get a general idea of how the server functions should be structured.

前面几个函数是涉及到文件系统到硬盘的管理，下面的则是完成用户的请求。`serve_read`函数接收用户进程的Fsipc

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

spawn的话可以从用户态加载应用程序去运行。

> **Exercise 9.** In your kern/trap.c, call kbd_intr to handle trap IRQ_OFFSET+IRQ_KBD and serial_intr to handle trap IRQ_OFFSET+IRQ_SERIAL. 

`kbd_intr`函数读取键盘的输入；`serial_intr`函数读取输入的数据到buffer里面。

```c
		case IRQ_OFFSET+IRQ_KBD:{
			kbd_intr();
			lapic_eoi();
			return;
		}
```

> **Exercise 10.** The shell doesn't support I/O redirection. It would be nice to run sh <script instead of having to type in all the commands in the script by hand, as you did above. Add I/O redirection for < to user/sh.c.

io重定向：以重定向输入为例（比如`sh<script`）。因为fd 0代表着stdin，fd 1代表着stdout，所以将待重定向的文件的fd拷贝到子进程的fd 0即可（执行`sh`的shell可以通过`getchar`的方式读取该文件内容【根据fd绑定的内容找到对应的`devfile`，那么`getchar`的读取转换成`devfile->read`】）。

```c
			if ((fd = open(t, O_RDONLY)) < 0) {
				cprintf("open %s for read: %e", t, fd);
				exit();
			}
			if(fd != 0){
				dup(fd,0);
				close(fd);
			}
			break;
```

