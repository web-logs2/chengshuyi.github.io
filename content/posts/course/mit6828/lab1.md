---
title: "Lab1 - 实验笔记"
date: 2020-04-24T20:24:40+08:00
description: ""
draft: flase
tags: [mit6828 OS实验]
categories: [mit6828 OS实验]
---

[lab1的实验代码](https://github.com/chengshuyi/jos-lab/commit/487630546efdc7ecc668ced4508cf6f164c56fd6)

### Getting Started with x86 assembly

NASM uses the so-called ***Intel* syntax** while GNU uses the ***AT&T* syntax.** [Brennan's Guide to Inline Assembly](http://www.delorie.com/djgpp/doc/brennan/brennan_att_inline_djgpp.html)

### The PC's Physical Address Space

```c

+------------------+  <- 0xFFFFFFFF (4GB)
|      32-bit      |
|  memory mapped   |
|     devices      |
|                  |
/\/\/\/\/\/\/\/\/\/\

/\/\/\/\/\/\/\/\/\/\
|                  |
|      Unused      |
|                  |
+------------------+  <- depends on amount of RAM
|                  |
|                  |
| Extended Memory  |
|                  |
|                  |
+------------------+  <- 0x00100000 (1MB)
|     BIOS ROM     |
+------------------+  <- 0x000F0000 (960KB)
|  16-bit devices, |
|  expansion ROMs  |
+------------------+  <- 0x000C0000 (768KB)
|   VGA Display    |
+------------------+  <- 0x000A0000 (640KB)
|                  |
|    Low Memory    |
|                  |
+------------------+  <- 0x00000000
```

* 早期的16位的8088处理器能够访问1MB的地址空间，可以知道对应地址总线是20位，数据总线是16位，对应的寄存器也就是16位的，所以intel采用segment:offset来计算物理地址[^2]。

* VGA(video display buffers)：视频显示用的，每隔一定的时间，对应的硬件就会从该区域读取要显示的图像。
* ROMs：存放固件。

* BIOS ROM：一般是nor flash，nor flash支持XIP(eXecute In Place)，可以将其数据映射到RAM地址空间。
* Extended Memory：高于1MB的剩余物理空间，最多到4GB。

>  JOS只能使用前256MB的物理内存？

### bootloader系统启动

下面是biod->bootloader->kernel entry的大致流程：

1. ROM BIOS开机自检，其第一个指令是`[f000:fff0] 0xffff0:	ljmp   $0xf000,$0xe05b`；

2. ROM BIOS将我们的启动代码加载到物理地址`0000:7c00`，最后`jmp`指令跳转到我们的代码处执行；

3. 处理器启动时进入real模式，并设置`cs=0xf000`和`ip=0xfff0`。物理地址是：`physical address = 16 * segment + offset`；

4. Enable A20[^2]：80286（24根地址线）为了兼容8088（20根地址线）；

   >8086所能访问的物理地址空间：0(0x0000:0x0000)到0x10FFEF(FFFF:FFFF)，多出来了第20位，因为8086地址线只有20根，所以没有影响；
   >
   >但是在80286中，Intel把地址线扩展成24根了，FFFF:FFFF真的就是0x10FFEF了，你让那些legacy software怎么活？！本来人家想读0xFFEF的，怎么成了0x10FFEF？不是人家不想好好工作，是你硬件设计的不让人家好好工作嘛。
   >
   >乃们看出来了没？A20 是 80286 时代照顾8088软件的产物。通常所说的32位保护模式是 80386 才出现的，**所以，A20跟保护模式毛关系都没有！**开不开都一样进，影响的只是第20位而已

5. 切换模式：从实模式切换到保护模式，保护模式使用gdt和ldt，可以访问到比实模式更多的物理地址空间；

6. 根据elf的信息将内核代码从硬盘加载到物理内存中；

   IO端口地址[^1]：

    ```c
    I/O Address	USE
    0010-001F	System
    01F0-01F7	IDE interface - Primary channel
    ```

**习题：**

- At what point does the processor start executing 32-bit code? What exactly causes the switch from 16- to 32-bit mode?

  ```c
  orl     $CR0_PE_ON, %eax
  movl    %eax, %cr0
  ljmp    $PROT_MODE_CSEG, $protcseg
  ```

  上面的代码使能保护模式，将cr0寄存器的第一位置1；第三行代码实在32bit模式下运行的。

- What is the *last* instruction of the boot loader executed, and what is the *first* instruction of the kernel it just loaded?

  `((void (*)(void)) (ELFHDR->e_entry))();`

  运行`readelf -h obj/kern/kernel`可以知道kernel的entry是0x1000c，所以第一条指令是：`f010000c:	66 c7 05 72 04 00 00 	movw   $0x1234,0x472`。

  > 这里的0xf010000c是虚拟地址（线性地址），0x1000c是物理地址；bootloader将内核加载到线性地址，这时还没有开启mmu，所以在开启mmu之前的代码必须是位置无关代码；如果要访问位置相关的代码，需要使用RELOC计算出线性地址对应的物理地址；
  >
  > 	kernel.ld
  > 	/* Link the kernel at this address: "." means the current address */
  > 	. = 0xF0100000;
  > 	
  > 	/* AT(...) gives the load address of this section, which tells
  > 	   the boot loader where to load the kernel in physical memory */
  > 	.text : AT(0x100000) {
  > 		*(.text .stub .text.* .gnu.linkonce.t.*)
  > 	}

- *Where* is the first instruction of the kernel?

  `0x1000c`

- How does the boot loader decide how many sectors it must read in order to fetch the entire kernel from disk? Where does it find this information?

  kernel最终被编译成elf文件，elf头部信息包含代码的详细信息。

### 内核启动流程

下面是内核汇编代码的大致运行流程：

1. 此时分页系统还未开启，内核访问位置相关的代码（比如数据），需要使用RELOC，手动将虚拟地址转换成物理地址；

2. 将页目录表基地址加载到cr3寄存器，再修改cr0寄存器，开启分页系统；

   ```c
   pde_t entry_pgdir[NPDENTRIES] = {
   	// Map VA's [0, 4MB) to PA's [0, 4MB)
   	[0]
   		= ((uintptr_t)entry_pgtable - KERNBASE) + PTE_P,
   	// Map VA's [KERNBASE, KERNBASE+4MB) to PA's [0, 4MB)
   	[KERNBASE>>PDXSHIFT]
   		= ((uintptr_t)entry_pgtable - KERNBASE) + PTE_P + PTE_W
   };
   讲一下为什么要映射VA[0,4MB)，因为当前的ip处于[0,4MB)，所以开启分页系统之后ip仍然处于[0,4MB)，如果没有VA[0,4MB)的映射会出现page fault。                                                     
   ```

3. 修改ip的值到内核空间；

   ```c
   	mov	$relocated, %eax
   	jmp	*%eax
   relocated:
   	...
   编译的时候都是按虚拟地址来进行编译的，所以relocated保存的是虚拟地址。
   ```

4. 进入c语言代码；

**习题：**

* Explain the interface between `printf.c` and `console.c`. Specifically, what function does `console.c` export? How is this function used by `printf.c`?

  printf库解析传进来的参数转换成字符串，并一个个的传给console；

  console将字符串打印出来，包括串口和显示屏显示；

* For the following questions you might wish to consult the notes for Lecture 2. These notes cover GCC's calling convention on the x86.

  Trace the execution of the following code step-by-step:

  ```c
  int x = 1, y = 3, z = 4;
  cprintf("x %d, y %x, z %d\n", x, y, z);
  ```

  - In the call to `cprintf()`, to what does `fmt` point? To what does `ap` point?
  - List (in order of execution) each call to `cons_putc`, `va_arg`, and `vcprintf`. For `cons_putc`, list its argument as well. For `va_arg`, list what `ap` points to before and after the call. For `vcprintf` list the values of its two arguments.

  > 我们知道再x86中参数都通过栈传递（现在一部分是通过通用寄存器传递的），而参数压栈的顺序是从右到左，所以fmt是指向上一个函数的栈顶，也就是`0x8(%ebp)`。ap指向fmt的前一个参数，也就是lea    0xc(%ebp),%eax。具体看下面的代码：

  ```assembly
  int
  cprintf(const char *fmt, ...)
  {
  f0100ad1:	55                   	push   %ebp
  f0100ad2:	89 e5                	mov    %esp,%ebp
  f0100ad4:	83 ec 10             	sub    $0x10,%esp
  	va_list ap;
  	int cnt;
  
  	va_start(ap, fmt);
  f0100ad7:	8d 45 0c             	lea    0xc(%ebp),%eax   // 取参数（除了fmt参数）的首地址
  	cnt = vcprintf(fmt, ap);
  f0100ada:	50                   	push   %eax
  f0100adb:	ff 75 08             	pushl  0x8(%ebp)		// 取值，该值指向fmt的字符串首地址
  f0100ade:	e8 b7 ff ff ff       	call   f0100a9a <vcprintf>
  	va_end(ap);
  
  	return cnt;
  }
  ```

* Let's say that GCC changed its calling convention so that it pushed arguments on the stack in declaration order, so that the last argument is pushed last. How would you have to change `cprintf` or its interface so that it would still be possible to pass it a variable number of arguments?

### The Stack

下面是栈的大概分布情况：

```c
		       +------------+   |
		       | arg 2      |   \
		       +------------+    >- previous function's stack frame
		       | arg 1      |   /
		       +------------+   |
		       | ret %eip   |   /
		       +============+   
		       | saved %ebp |   \
		%ebp-> +------------+   |
		       |            |   |
		       |   local    |   \
		       | variables, |    >- current function's stack frame
		       |    etc.    |   /
		       |            |   |
		       |            |   |
		%esp-> +------------+   /
```

根据发生错误的ip输出backtrace：通过链接脚本，将程序的符号表加载到elf文件中，具体可以参看[stabs](http://sourceware.org/gdb/onlinedocs/stabs.html)。

### 参考链接

[^1]: IO端口地址分布, http://web.archive.org/web/20040304063834/http://members.iweb.net.au/~pstorr/pcbook/book2/ioassign.htm
[^2]: OS boot 的时候为什么要 enable A20？ - 坂本鱼子酱的回答 - 知乎 https://www.zhihu.com/question/29375534/answer/44137152