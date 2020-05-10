---
title: "Linux用户态设备驱动及dpdk和spdk"
date: 2020-05-04T07:02:51+08:00
description: ""
draft: true
tags: [linux内核]
categories: [linux内核]
---

本文主要介绍了linux用户态设备驱动的起源和实现，详细解析了uio驱动源码。根据lwn上关于用户态设备驱动的几篇文章，阐述了用户态设备驱动实现的几个难点，以及如何实现。

### 用户态设备驱动的目的

在[^1]中提出了以下几个在用户态实现设备驱动所具有的优点：

1. 用户态设备驱动更简单；
    a. 用户态的代码调试要比调试内核简单许多；
    b. 用户态不需要关系内核设备驱动框架，比如注册各种设备文件等等；
2. 创建更稳定的ABI；
3. 解决license问题，用户态实现的驱动不受linux GPL license限制。
4. 可以用其它的语言编写驱动，比如python；

但是，也有人提出用户态驱动使得linux内核看起来像微内核，而且不符合开源精神。

### 实现用户态设备驱动的难点

我们知道一个设备驱动是管理设备的接口，一个设备通常具有几个重要的属性：io端口或者mmio的内存地址空间、中断、DMA和PCI等。下面分别介绍如何将这几个属性的权限移交给用户态：

1. io端口的访问：linux允许特权级进程通过I/O指令；

```c
//code from: http://www.faqs.org/docs/Linux-mini/IO-Port-Programming.html
/*
 * example.c: very simple example of port I/O
 *
 * This code does nothing useful, just a port write, a pause,
 * and a port read. Compile with `gcc -O2 -o example example.c',
 * and run as root with `./example'.
 */

#include <stdio.h>
#include <unistd.h>
#include <asm/io.h>

#define BASEPORT 0x378 /* lp1 */

int main()
{
  /* Get access to the ports */
  if (ioperm(BASEPORT, 3, 1)) {perror("ioperm"); exit(1);}
  
  /* Set the data signals (D0-7) of the port to all low (0) */
  outb(0, BASEPORT);
  
  /* Sleep for a while (100 ms) */
  usleep(100000);
  
  /* Read from the status port (BASE+1) and display the result */
  printf("status: %d\n", inb(BASEPORT + 1));

  /* We don't need the ports anymore */
  if (ioperm(BASEPORT, 3, 0)) {perror("ioperm"); exit(1);}

  exit(0);
}

/* end of example.c */
```

2. mmio的访问：可以通过`/dev/mem`文件提供的`mmap`函数做映射，将设备地址空间映射到进程地址空间中。这个方法已经集成到uio里面了。

3. ioctl /proc文件，可以让用户访问PCI的配置空间；todo

4. 中断：[User mode drivers: part 1, interrupt handling](https://lwn.net/Articles/127293/)，该patch解决了用户驱动如何获取中断的问题：

    a. 在`/proc/irq/`创建中断的设备文件，提供了四个方法，分别是：`read`、`open`、`release`和`poll`；

    ```c
    static struct file_operations irq_proc_file_operations = {
     	.read = irq_proc_read,
     	.open = irq_proc_open,
     	.release = irq_proc_release,
     	.poll = irq_proc_poll,
    };
    ```
    b. `open`函数为用户进程初始化`wait`队列，并向该中断号注入handler，当中断发生时会执行该handler；
    c. `read`函数将该用户进程加入到`wait`队列，将该进程状态设置为`TASK_INTERRUPTIBLE`，发起调度；
    d. 当中断发生时执行b中注入的handler，该handler会从`wait`队列中唤醒等待进程；
    e. `read`函数返回；
    f. 也可以使用`poll`的方式。

5. DMA：提供两个新的调用

```c
int usr_pci_open(int bus, int slot, int function);
int usr_pci_map(int fd, int cmd, struct mapping_info *info);
```


### UIO

### dpdk

### spdk

将Linux变得看起来有点像微内核   可以使用二进制驱动
https://lwn.net/Articles/66829/
http://www.gelato.unsw.edu.au/IA64wiki/UserLevelDrivers

the patch https://lwn.net/Articles/127293/

### 参考文章

[1]. [lwn article: User-space device drivers](https://lwn.net/Articles/66829/)


```c
struct uio_info {
	struct uio_device	*uio_dev;
	const char		*name;
	const char		*version;
	struct uio_mem		mem[MAX_UIO_MAPS];
	struct uio_port		port[MAX_UIO_PORT_REGIONS];
	long			irq;
	unsigned long		irq_flags;
	void			*priv;
	irqreturn_t (*handler)(int irq, struct uio_info *dev_info);
	int (*mmap)(struct uio_info *info, struct vm_area_struct *vma);
	int (*open)(struct uio_info *info, struct inode *inode);
	int (*release)(struct uio_info *info, struct inode *inode);
	int (*irqcontrol)(struct uio_info *info, s32 irq_on);
};

static const struct file_operations uio_fops = {
	.owner		= THIS_MODULE,
	.open		= uio_open,
	.release	= uio_release,
	.read		= uio_read,
	.write		= uio_write,
	.mmap		= uio_mmap,
	.poll		= uio_poll,
	.fasync		= uio_fasync,
	.llseek		= noop_llseek,
};
```



https://github.com/torvalds/linux/commit/beafc54c4e2fba24e1ca45cdb7f79d9aa83e3db1#diff-f8c4bf947cbbef4f6cc9d168af57c347

idev->dev = device_create(uio_class->class, parent,
				  MKDEV(uio_major, idev->minor),
				  "uio%d", idev->minor);

ret = request_irq(idev->info->irq, uio_interrupt, idev->info->irq_flags, idev->info->name, idev);