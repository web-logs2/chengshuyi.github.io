---
title: "eBPF 辅助 (helper) 函数设计与实现"
date: 2023-04-22T14:08:58+08:00 
description: ""
draft: false
tags: [eBPF,eBPF 辅助函数]
categories: [eBPF]
---


您是否想为内核添加一个新的 eBPF 辅助函数，但不知道从何入手？或者，您是否曾遇到过类似于 `R2 type=ctx expected=fp, pkt, pkt_meta, map_value` 的 eBPF verifier 报错？本文将从代码层面对 eBPF 辅助函数在内核中的设计与实现进行深入浅出的分析。相信在阅读本文后，您不仅能够轻松应对由于错误调用辅助函数导致的 eBPF verifier 问题，还能了解如何实现一个新的 eBPF 辅助函数。

本文首先简单介绍了 eBPF 辅助函数的概念，并探讨了其在内核中的设计，包括哪些重要组成部分。随后，结合辅助函数 `bpf_perf_event_output` 的实现，帮助读者了解实现一个 eBPF 辅助函数所需的要素。最后，我们通过一段代码分析了调用辅助函数时传入不匹配的参数类型导致 eBPF verifier 报错的问题。


## 简介

什么是 eBPF 辅助函数？eBPF 辅助函数是内核提供给开发者的接口。

为什么要有 eBPF 辅助函数呢？为什么不能像驱动一样直接调用内核函数呢？ 这主要是为了保证系统安全。由于 eBPF 程序运行在内核态，为了防止不当调用内核函数导致系统崩溃或安全漏洞，eBPF 程序只能调用内核提供的 eBPF 辅助函数。

截止目前内核共提供了 210 多个 eBPF 辅助函数，具体详细列表可见内核源码文件：`include/uapi/linux/bpf.h`。


## eBPF 辅助函数的设计


在内核中，`struct bpf_func_proto` 描述了 eBPF 辅助函数的定义、入参类型、返回值类型等重要信息。这些信息的指定主要是为了通过 eBPF verifier 的安全验证，确保传入数据的可靠性，避免传入错误的参数导致系统崩溃。`struct bpf_func_proto` 的具体形式的代码片段如下所示：

```c
struct bpf_func_proto {
	//eBPF 辅助函数具体实现
	u64 (*func)(u64 r1, u64 r2, u64 r3, u64 r4, u64 r5);
	bool gpl_only;
	bool pkt_access;
	bool might_sleep;
	// 返回类型
	enum bpf_return_type ret_type;
	union {
		// 参数类型
		struct {
			enum bpf_arg_type arg1_type;
			enum bpf_arg_type arg2_type;
			enum bpf_arg_type arg3_type;
			enum bpf_arg_type arg4_type;
			enum bpf_arg_type arg5_type;
		};
		enum bpf_arg_type arg_type [5];
	};
	union {
		// 当参数类型为 ARG_PTR_TO_BTF_ID，需要指明参数的 BTF 编号
		struct {
			u32 *arg1_btf_id;
			u32 *arg2_btf_id;
			u32 *arg3_btf_id;
			u32 *arg4_btf_id;
			u32 *arg5_btf_id;
		};
		u32 *arg_btf_id [5];
		struct {
			size_t arg1_size;
			size_t arg2_size;
			size_t arg3_size;
			size_t arg4_size;
			size_t arg5_size;
		};
		size_t arg_size [5];
	};
	// 返回参数的 BTF 编号
	int *ret_btf_id;
	bool (*allowed)(const struct bpf_prog *prog);
};
```

其中，`func` 表示该 eBPF 辅助函数的具体实现，实现了特定的功能。`bpf_return_type` 描述该 eBPF 辅助函数的返回参数类型，而 `argx_type` 描述该函数的入参类型。下面将对入参类型和返回值类型进行解析。

### 入参类型

入参类型分为基本类型和扩展类型。扩展类型在基本类型的基础上，添加了空指针类型，即允许入参为空指针。另外，当参数类型为 `ARG_PTR_TO_BTF_ID` 时，则需要在 `struct bpf_func_proto` 的成员 `argx_btf_id` 指明具体的 btf 编号。

**注：BTF 编号可以看成内核数据类型的编号，通过该编号可以确定数据类型。**

#### 基本类型

基本类型大致包含三类：

1. 指针类型，指针类型又可以进行细分：1）具体类型的指针类型，如 `ARG_PTR_TO_SOCKET` 表示 `struct socket` 指针；2）由 BTF 编号确定数据类型的指针类型，如 `ARG_PTR_TO_BTF_ID` 表示某一内核数据类型指针，且该内核数据类型由 BTF 编号指定；3）指向某一类型内存的指针，如 `ARG_PTR_TO_MAP_KEY` 指向 eBPF 程序栈内存的指针。
2. 整数类型，如 `ARG_CONST_SIZE` 表示整数，且该整数的值不能为 0；
3. 任意类型，即 `ARG_ANYTHING`，其表示任意类型，但是需要初始化该值，否则 eBPF verifier 会报 `未初始化` 等相关错误。


完整的基本类型如下表所示：

| 类型 | 含义 |
| ----- | ---- |
| ARG_CONST_MAP_PTR | `struct bpf_map` 指针，比如调用辅助函数 `bpf_map_lookup_elem` 第一个参数就是 `struct bpf_map` 指针 |
| ARG_PTR_TO_MAP_KEY | 指向 eBPF 程序栈内存的指针，且该内存的数据是 map 的键 |
| ARG_PTR_TO_MAP_VALUE | eBPF map 的元素值指针 |
| ARG_PTR_TO_MEM | 指向栈、报文或 eBPF map 元素值的指针 |
| ARG_CONST_SIZE | 整数且值不为 0 |
| ARG_CONST_SIZE_OR_ZERO | 整数 |
| ARG_CONST_ALLOC_SIZE_OR_ZERO | 整数，虽然和 ARG_CONST_SIZE_OR_ZERO 同表示整数，但是语义不同 |
| ARG_PTR_TO_CTX | `struct pt_regs` 指针 |
| ARG_ANYTHING | 表示任意类型，但是其值需要初始化 |
| ARG_PTR_TO_SPIN_LOCK | `struct bpf_spin_lock` 指针 |
| ARG_PTR_TO_SOCK_COMMON | `struct sock_common` 指针 |
| ARG_PTR_TO_INT | `int` 指针 |
| ARG_PTR_TO_LONG |`long` 指针 |
| ARG_PTR_TO_SOCKET | `struct socket` 指针 |
| ARG_PTR_TO_BTF_ID | 内核任意结构体指针，`ARG_PTR_TO_SOCKET` 是特指 `struct socket` 指针，而 `ARG_PTR_TO_BTF_ID` 表示内核任意结构体指针，且该指针用 BTF 编号表示 |
| ARG_PTR_TO_RINGBUF_MEM | ring buffer 保留内存指针 |
| ARG_PTR_TO_BTF_ID_SOCK_COMMON |`struct sock_common` 或 `struct bpf_sock` 指针 |
| ARG_PTR_TO_PERCPU_BTF_ID | 指向 percpu 类型的指针，其中 percpu 具体类型由 BTF 编号指定 |
| ARG_PTR_TO_FUNC | eBPF 程序指针 |
| ARG_PTR_TO_STACK | 指向 eBPF 程序栈内存的指针 |
| ARG_PTR_TO_CONST_STR | 字符串指针 |
| ARG_PTR_TO_TIMER | `struct bpf_timer` 指针 |
| ARG_PTR_TO_KPTR | kptr 类型指针，kptr 表示指向内核态地址空间的指针 |
| ARG_PTR_TO_DYNPTR | `struct bpf_dynptr` 指针 |


#### 扩展类型

包含的扩展类型如下表所示：

| 类型 | 含义 |
| ----- | ---- |
| ARG_PTR_TO_MAP_VALUE_OR_NULL | eBPF map 元素值指针 |
| ARG_PTR_TO_MEM_OR_NULL | 指向栈、报文或 eBPF map 元素值的指针或者空指针 |
| ARG_PTR_TO_CTX_OR_NULL | `struct pt_regs` 指针或者空指针 |
| ARG_PTR_TO_SOCKET_OR_NULL | `struct socket` 指针或者空指针 |
| ARG_PTR_TO_STACK_OR_NULL | 指向 eBPF 程序栈内存的指针或者空指针 |
| ARG_PTR_TO_BTF_ID_OR_NULL | 该类型由 BTF 编号指定，且允许传入的值为空 |
| ARG_PTR_TO_UNINIT_MEM | 指向内存空间的指针，允许该片内存未初始化 |
| ARG_PTR_TO_FIXED_SIZE_MEM | 指向固定内存空间的指针，且在编译期间能够确定该内存大小 |

### 返回值类型

同参数类型类似，返回值类型也分为基本类型和扩展类型。扩展类型也是在基本类型的基础添加了空指针类型。

#### 基本类型

| 返回值类型 | 描述 |
| ---- | ---- |
| RET_INTEGER | 返回整数类型 |
| RET_VOID | 无返回值 |
| RET_PTR_TO_MAP_VALUE | eBPF map 元素值的指针 |
| RET_PTR_TO_SOCKET | `struct socket` 指针 |
| RET_PTR_TO_TCP_SOCK | `struct tcp_sock` 指针 |
| RET_PTR_TO_SOCK_COMMON | `struct sock_common` 指针 |
| RET_PTR_TO_MEM | 指向有效内存的指针 |
| RET_PTR_TO_MEM_OR_BTF_ID | 指向有效内存的指针或者 BTF 编号 |
| RET_PTR_TO_BTF_ID | 内核任意结构体指针，由 BTF 编号表示具体类型 |

#### 扩展类型

扩展类型是在基本类型的基础上，添加了空指针类型，表示返回值可能是空指针，那么 eBPF verifier 需要考虑针对空指针进行安全验证。

| 返回值类型 | 描述 |
| ---- | ---- |
| RET_PTR_TO_MAP_VALUE_OR_NULL | eBPF map 元素值的指针或者空指针，如 eBPF 辅助函数 `bpf_map_lookup_elem` 使用其作为返回值类型 |
| RET_PTR_TO_SOCKET_OR_NULL | `struct socket` 指针或空指针 |
| RET_PTR_TO_TCP_SOCK_OR_NULL | `struct tcp_sock` 指针或空指针 |
| RET_PTR_TO_SOCK_COMMON_OR_NULL | `struct sock_common` 指针或空指针 |
| RET_PTR_TO_RINGBUF_MEM_OR_NUL | ring buffer 的内存指针或空指针 |
| RET_PTR_TO_DYNPTR_MEM_OR_NULL | `struct bpf_dynptr` 指针或空指针 |
| RET_PTR_TO_BTF_ID_OR_NULL | 该类型由 BTF 编号指定，且允许传入的值为空 |
| RET_PTR_TO_BTF_ID_TRUSTED | 表示该指针是安全的，且该指针类型由 BTF 编号确定 |


## eBPF 辅助函数的实现

本小节以 `bpf_perf_event_output` 为例介绍 eBPF 辅助函数的实现。eBPF 辅助函数 `bpf_perf_event_output` 是应用最广泛的一个，其主要功能是将数据通过 perf 缓冲区传送给用户态程序。实现 `bpf_perf_event_output` 需要完成以下三个步骤：

1. 定义 `struct bpf_func_proto` 结构体，为 `bpf_perf_event_output` 辅助函数指定功能函数、参数类型、返回值类型等；
2. 为 `bpf_perf_event_output` 辅助函数分配唯一的编号；
3. 将 `bpf_perf_event_output` 与特定的 eBPF 程序类型绑定，以确保只有该类型的程序才能调用该辅助函数。

### 定义 struct bpf_func_proto

```c
BPF_CALL_5 (bpf_perf_event_output, struct pt_regs *, regs, struct bpf_map *, map,
	   u64, flags, void *, data, u64, size)
{
	......
	return err;
}

static const struct bpf_func_proto bpf_perf_event_output_proto = {
	.func		= bpf_perf_event_output,
	.gpl_only	= true,
	.ret_type	= RET_INTEGER,
	.arg1_type	= ARG_PTR_TO_CTX,
	.arg2_type	= ARG_CONST_MAP_PTR,
	.arg3_type	= ARG_ANYTHING,
	.arg4_type	= ARG_PTR_TO_MEM | MEM_RDONLY,
	.arg5_type	= ARG_CONST_SIZE_OR_ZERO,
};
```

`bpf_perf_event_output` 的入参类型分别是：

1. `ARG_PTR_TO_CTX`: `struct pt_regs` 指针
2. `ARG_CONST_MAP_PTR`: `struct bpf_map` 指针
3. `ARG_ANYTHING`：任意类型，且数值已初始化
4. `ARG_PTR_TO_MEM | MEM_RDONLY`: 指向栈、报文或 eBPF map 元素值的指针
5. `ARG_CONST_SIZE_OR_ZERO`: 整数且该整数值可为 0

返回值类型是整数类型：`RET_INTEGER`


### 添加编号


在完成 `struct bpf_func_proto` 的定义之后，需要为其分配一个唯一的编号。下面的代码片段通过将其扩展为 `BPF_FUNC_perf_event_output` 宏定义，并将该辅助函数的编号设置为 25，即 `#define BPF_FUNC_perf_event_output 25`。

**注：该代码片段位于内核源文件：`include/uapi/linux/bpf.h`**

```c
#define ___BPF_FUNC_MAPPER (FN, ctx...)	
	FN (unspec, 0, ##ctx)				\
	......
	FN (perf_event_output, 25, ##ctx)		\
	......
```

### 绑定 eBPF 程序类型

最后一步是要指定允许调用该辅助函数的 eBPF 程序类型。例如，下面的代码片段中，允许 `BPF_PROG_TYPE_KPROBE` 类型的 eBPF 程序调用 `bpf_perf_event_output` 辅助函数。如果未指定允许调用该辅助函数的程序类型的 eBPF 程序调用了该辅助函数，则在 eBPF 程序加载过程会出现类似于 `unknown func bpf_perf_event_output#25` 的 eBPF verifier 错误提示。

```c
static const struct bpf_func_proto *
kprobe_prog_func_proto (enum bpf_func_id func_id, const struct bpf_prog *prog)
{
	switch (func_id) {
	case BPF_FUNC_perf_event_output:
		return &bpf_perf_event_output_proto;
	......
	default:
		return bpf_tracing_func_proto (func_id, prog);
	}
}
const struct bpf_verifier_ops kprobe_verifier_ops = {
	.get_func_proto  = kprobe_prog_func_proto,  // 验证改类型的 eBPF 程序是否可调用 func_id 所代表的辅助函数
	.is_valid_access = kprobe_prog_is_valid_access,
};
```

## 小试牛刀

在理解了上述的理论知识后，我们可以来看看如何定位并解决开篇提到的问题：`R2 type=ctx expected=fp, pkt, pkt_meta, map_value`。下面是引起该错误的代码示例，读者可以分析该代码存在哪些问题以及如何解决这些问题。

```c
struct
{
    __uint (type, BPF_MAP_TYPE_HASH);
    __type (key, struct sock *);
    __type (value, struct sockmap_val);
    __uint (max_entries, 1024);
} sockmap SEC (".maps");

struct sockmap_val
{
    int nothing;
};

SEC ("tracepoint/tcp/tcp_rcv_space_adjust")
int tp__tcp_rcv_space_adjust (struct trace_event_raw_tcp_event_sk *ctx)
{
    struct sockmap_val *sv = bpf_map_lookup_elem (&sockmap, &ctx->skaddr);
    if (sv)
        bpf_printk ("% d\n", sv->nothing);
    return 0;
}
```

### 问题解析

首先解释一下错误信息 `R2 type=ctx expected=fp, pkt, pkt_meta, map_value` 的含义。该错误表示 R2 寄存器的数据类型应该是指向栈内存的指针、报文指针、或者 eBPF map 的元素值指针，但实际数据类型是 ctx，即指向 `struct pt_regs` 的指针。因此，该问题实际上是因为数据类型不匹配引起的。

在调用 `bpf_map_lookup_elem (&sockmap, &ctx->skaddr)` 函数时，我们传递的参数 `&ctx->skaddr` 是 ctx 类型参数，而不是 fp 类型参数。那么为什么会有这个限制呢？

根据上文所述，eBPF 辅助函数的入参类型是通过 `struct bpf_func_proto` 进行定义的。我们可以参考 `bpf_map_lookup_elem` 辅助函数在内核代码中的实现来解释这个问题。在该函数的代码片段中，可以看到它的第二个入参类型为 `ARG_PTR_TO_MAP_KEY`，即指向 eBPF 程序栈内存的指针，也就是 fp。

```c
const struct bpf_func_proto bpf_map_lookup_elem_proto = {
	.func		= bpf_map_lookup_elem,
	.gpl_only	= false,
	.pkt_access	= true,
	.ret_type	= RET_PTR_TO_MAP_VALUE_OR_NULL,
	.arg1_type	= ARG_CONST_MAP_PTR,
	.arg2_type	= ARG_PTR_TO_MAP_KEY,
};
```

### 解决方案

针对这个问题，一般的解决方法是先定义一个栈变量，将 `ctx->skaddr` 的值存储到栈上，例如 `u64 skaddr = ctx->skaddr`，然后在调用 `bpf_map_lookup_elem` 函数时，将该栈变量的地址 `&skaddr` 作为函数的参数传递进去。

## 总结

本文重点介绍了 eBPF 辅助函数在内核中的设计，并描述了参数类型、返回值类型等重要概念。以 `bpf_perf_event_output` 为例，介绍了实现一个 eBPF 辅助函数的核心要素。eBPF 辅助函数在开发 eBPF 程序中扮演着重要的角色，深入地了解 eBPF 辅助函数的设计和实现可以帮助解决开发过程中的许多相关问题。