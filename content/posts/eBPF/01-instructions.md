---
title: "eBPF 指令集分析和总结"
date: 2023-04-22T14:08:58+08:00 
description: ""
draft: true
tags: [kvm,嵌入式虚拟化,arm]
categories: [kvm,嵌入式虚拟化]
---






## 跳转（JMP）指令


### 无条件跳转指令

其code是BPF_JA，具体形式是pc += off

```c
#define BPF_JMP_A(OFF)						\
	((struct bpf_insn) {					\
		.code  = BPF_JMP | BPF_JA,			\
		.dst_reg = 0,					\
		.src_reg = 0,					\
		.off   = OFF,					\
		.imm   = 0 })
```

### 条件跳转指令


```c
#define BPF_JMP_REG(OP, DST, SRC, OFF)				\
	((struct bpf_insn) {					\
		.code  = BPF_JMP | BPF_OP(OP) | BPF_X,		\
		.dst_reg = DST,					\
		.src_reg = SRC,					\
		.off   = OFF,					\
		.imm   = 0 })

#define BPF_JMP_IMM(OP, DST, IMM, OFF)				\
	((struct bpf_insn) {					\
		.code  = BPF_JMP | BPF_OP(OP) | BPF_K,		\
		.dst_reg = DST,					\
		.src_reg = 0,					\
		.off   = OFF,					\
		.imm   = IMM })

#define BPF_JMP32_REG(OP, DST, SRC, OFF)			\
	((struct bpf_insn) {					\
		.code  = BPF_JMP32 | BPF_OP(OP) | BPF_X,	\
		.dst_reg = DST,					\
		.src_reg = SRC,					\
		.off   = OFF,					\
		.imm   = 0 })

#define BPF_JMP32_IMM(OP, DST, IMM, OFF)			\
	((struct bpf_insn) {					\
		.code  = BPF_JMP32 | BPF_OP(OP) | BPF_K,	\
		.dst_reg = DST,					\
		.src_reg = 0,					\
		.off   = OFF,					\
		.imm   = IMM })
```



### 函数调用（CALL）指令

函数调用指令主要分为两大类，一类是eBPF程序调用eBPF程序自身内部的函数，即bpf2bpf调用，另外一类是eBPF程序调用外部的函数；

**bpf2bpf调用**

回想之前我们写的eBPF程序都必须加上always_inline，让它们打包成一个程序。主要是因为当时不支持bpf2bpf调用，引入bpf2bpf调用主要解决了两个问题：

 * 编译后的eBPF程序体积膨胀，

 * 导致用户需要把程序放到头文件里面才能重复使用


感兴趣的读者可以查看该功能的[内核补丁](https://github.com/torvalds/linux/commit/cc8b0b92a1699bc32f7fec71daa2bfc90de43a4d)。

bpf2bpf调用的源操作数是`BPF_PSEUDO_CALL`，立即数是目标函数首条指令相对偏移。具体形式如下所示：

```c
#define BPF_CALL_REL(TGT)					\
	((struct bpf_insn) {					\
		.code  = BPF_JMP | BPF_CALL,			\
		.dst_reg = 0,					\
		.src_reg = BPF_PSEUDO_CALL,			\
		.off   = 0,					\
		.imm   = TGT })
```

下面是一个具体的示例，其中`call pc+1`指令的源操作数是`BPF_PSEUDO_CALL`，立即数是1，调用了eBPF程序内部的函数。可以看到这里完全是函数调用的逻辑，不再是内联函数。

```c
r6 = r1    // some code
..
r1 = ..    // arg1
r2 = ..    // arg2
call pc+1  // function call pc-relative
exit
.. = r1    // access arg1
.. = r2    // access arg2
..
call pc+20 // second level of function call
...
```

**辅助函数调用**


辅助函数是eBPF提供给用户可调用的函数。由于eBPF程序运行在内核态，为了保证安全性，eBPF程序中不能随意调用内核函数，只能调用eBPF提供的辅助函数。如我们经常使用的`bpf_ktime_get_ns`获取时间戳就是一个辅助函数。截止目前为止共有211个辅助函数。完整的辅助函数列表定义在`include/uapi/linux/bpf.h`文件中


其具体的指令形式如下，源/目的操作数、偏移均为保留字段，立即数是调用的辅助函数编号。

```c
#define BPF_EMIT_CALL(FUNC)					\
	((struct bpf_insn) {					\
		.code  = BPF_JMP | BPF_CALL,			\
		.dst_reg = 0,					\
		.src_reg = 0,					\
		.off   = 0,					\
		.imm   = FUNC })

```


**eBPF kfunc调用**

eBPF 内核函数，即kfunc，是内核中公开供eBPF程序使用的函数。与一般的eBPF辅助函数不同，kfunc没有稳定的接口。此外，kfunc和程序类型是绑定的：register_btf_kfunc_id_set。

源操作数是`BPF_PSEUDO_KFUNC_CALL`，立即数是调用的内核函数的btf id。

BPF_RAW_INSN(BPF_JMP | BPF_CALL, 0, BPF_PSEUDO_KFUNC_CALL, 0, 0)

2021 [bpf: Support bpf program calling kernel function](https://lore.kernel.org/bpf/20210325015124.1543397-1-kafai@fb.com/t/)

官方文档：https://docs.kernel.org/bpf/kfuncs.html

特点：
1. 可调用的kfunc和program type绑定；
2. 作者举了个例子，bpf-tcp-cc 可以调用内核原有的一些函数；

**BPF_PSEUDO_FUNC**

目前场景看起来是用于在eBPF函数内，对map进行遍历：bpf_for_each_map_elem，将遍历函数的偏移保存到bpf_for_each_map_elem的参数中，以便后续的重定位。
可以看到该helper的第三个参数类型是：ARG_PTR_TO_FUNC。

大致思路是：你在eBPF函数调用bpf_for_each_map_elem，传入你的处理函数。那么当事件触发时，内核会遍历所有元素，每个元素执行你传入的处理函数。

由于这个不是直接函数调用方式，而是将函数指针传给eBPF，有内核的eBPF进行调用。

2021 引入patch：bpf: add bpf_for_each_map_elem() helper：https://lore.kernel.org/bpf/20210226204920.3884074-1-yhs@fb.com/


### 程序退出指令


```c
#define BPF_EXIT_INSN()						\
	((struct bpf_insn) {					\
		.code  = BPF_JMP | BPF_EXIT,			\
		.dst_reg = 0,					\
		.src_reg = 0,					\
		.off   = 0,					\
		.imm   = 0 })
```