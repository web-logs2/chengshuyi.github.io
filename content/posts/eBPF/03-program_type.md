---
title: "eBPF 程序类型（program types）分析和总结"
date: 2023-04-22T14:08:58+08:00 
description: ""
draft: true
tags: [eBPF,eBPF program types]
categories: [eBPF]
---


我们在编写eBPF程序时，需要再eBPF程序代码上添加类型的`SEC()`字段，这里的`SEC()`字段就指明了该eBPF的程序类型。如下代码片段，`SEC("kprobe/tcp_v4_connect")`表明了该eBPF程序的程序类型是`BPF_PROG_TYPE_KPROBE`。

```c
SEC("kprobe/tcp_v4_connect")
int BPF_KPROBE(tcp_v4_connect, struct sock *sk, struct sk_buff *skb)
{
    ......
    return 0;
}
```

对于部分程序类型来说，可能还存在装载类型（attach type），装载类型往往指定了该eBPF程序的装载点（内核位置）。

更多的可参考内核文档[eBPF程序类型和装载类型](https://docs.kernel.org/bpf/libbpf/program_types.html#tp)

不同的eBPF程序类型（program types）能够实现的功能不同，在实际的开发中，我们需要根据实际的需求去选择eBPF程序类型。目前内核共支持30余种程序类型，大致可分为如下几类：

* 跟踪、诊断、观测相关
* 

# eBPF 程序类型设计


如何设计并实现一个新的eBPF程序类型呢？其实，实现一个新的eBPF程序类型非常简单，只需要了解内核提供的3个结构体即可，分别是：

- struct bpf_verifier_ops：用于eBPF verifier相关功能实现；
- struct bpf_prog_ops：用于eBPF程序测试；
- struct bpf_prog_offload_ops：eBPF程序卸载到硬件所需功能；



## struct bpf_verifier_ops

`bpf_verifier_ops`主要用于eBPF verifier阶段，约束该程序类型的eBPF程序能够调用的辅助函数、内存访问等。核心成员有：



```c
struct bpf_verifier_ops {
	const struct bpf_func_proto *
	(*get_func_proto)(enum bpf_func_id func_id,
			  const struct bpf_prog *prog);
	bool (*is_valid_access)(int off, int size, enum bpf_access_type type,
				const struct bpf_prog *prog,
				struct bpf_insn_access_aux *info);
	int (*gen_prologue)(struct bpf_insn *insn, bool direct_write,
			    const struct bpf_prog *prog);
	int (*gen_ld_abs)(const struct bpf_insn *orig,
			  struct bpf_insn *insn_buf);
	u32 (*convert_ctx_access)(enum bpf_access_type type,
				  const struct bpf_insn *src,
				  struct bpf_insn *dst,
				  struct bpf_prog *prog, u32 *target_size);
	int (*btf_struct_access)(struct bpf_verifier_log *log,
				 const struct bpf_reg_state *reg,
				 int off, int size, enum bpf_access_type atype,
				 u32 *next_btf_id, enum bpf_type_flag *flag);
};
```


- bpf_func_proto

`bpf_func_proto`的原型是`const struct  bpf_func_proto* (*get_func_proto)(enum bpf_func_id func_id, const struct bpf_prog *prog)`。

其中，入参`func_id`是辅助函数编号，`bpf_func_proto`主要用于验证该程序类型的eBPF程序能否调用`func_id`辅助函数，并返回该辅助函数的原型定义。下面是`BPF_PROG_TYPE_KPROBE`程序类型对`bpf_func_proto`的实现。可以看到，`BPF_PROG_TYPE_KPROBE`程序类型允许调用`bpf_perf_event_output`、`bpf_get_stackid`、`bpf_get_stack`等辅助函数。

```c
static const struct bpf_func_proto *
kprobe_prog_func_proto(enum bpf_func_id func_id, const struct bpf_prog *prog)
{
	switch (func_id) {
	case BPF_FUNC_perf_event_output:
		return &bpf_perf_event_output_proto;
	case BPF_FUNC_get_stackid:
		return &bpf_get_stackid_proto;
	case BPF_FUNC_get_stack:
		return &bpf_get_stack_proto;
	default:
		// 定义了一些通用的可调用的辅助函数，如对eBPF map的操作：
		// bpf_map_lookup_elem
		// bpf_map_update_elem
		// ......
		return bpf_tracing_func_proto(func_id, prog);
	}
}
```


- `gen_prologue`

目前只有少部分的eBPF程序类型需要实现这个。
这里以BPF_PROG_TYPE_SK_SKB为例


	// CLONED_OFFSET指的是成员__cloned_offset在struct sk_buff中的偏移
	// #define CLONED_OFFSET		offsetof(struct sk_buff, __cloned_offset)

```c
static int sk_skb_prologue(struct bpf_insn *insn_buf, bool direct_write,
			   const struct bpf_prog *prog)
{
	return bpf_unclone_prologue(insn_buf, direct_write, prog, SK_DROP);
}

static int bpf_unclone_prologue(struct bpf_insn *insn_buf, bool direct_write,
				const struct bpf_prog *prog, int drop_verdict)
{
	struct bpf_insn *insn = insn_buf;

	if (!direct_write)
		return 0;
	
	// 1. 	r6 = *(u8 *)(r1 + CLONED_OFFSET)
	// 2.	r6 &= CLONED_MASK
	// 3.	if r6 == 0 goto pc+7
	*insn++ = BPF_LDX_MEM(BPF_B, BPF_REG_6, BPF_REG_1, CLONED_OFFSET);
	*insn++ = BPF_ALU32_IMM(BPF_AND, BPF_REG_6, CLONED_MASK);
	*insn++ = BPF_JMP_IMM(BPF_JEQ, BPF_REG_6, 0, 7);
	// 4.	r6 = r1
	// 5.	r2 ^= r2
	// 6.	call bpf_skb_pull_data
	*insn++ = BPF_MOV64_REG(BPF_REG_6, BPF_REG_1);
	*insn++ = BPF_ALU64_REG(BPF_XOR, BPF_REG_2, BPF_REG_2);
	*insn++ = BPF_RAW_INSN(BPF_JMP | BPF_CALL, 0, 0, 0,
			       BPF_FUNC_skb_pull_data);
	// 7.	if r0 == 0 goto pc+2
	// 8.	r0 = drop_verdict
	// 9.	exit
	*insn++ = BPF_JMP_IMM(BPF_JEQ, BPF_REG_0, 0, 2);
	*insn++ = BPF_ALU32_IMM(BPF_MOV, BPF_REG_0, drop_verdict);
	*insn++ = BPF_EXIT_INSN();
	// 10.	restore: r1 = r6
	*insn++ = BPF_MOV64_REG(BPF_REG_1, BPF_REG_6);
	// 11.	start
	*insn++ = prog->insnsi[0];

	return insn - insn_buf;
}
```

- `gen_ld_abs`



- `convert_ctx_access`


- `btf_struct_access`


## struct bpf_prog_ops

主要用于实现测试。

```c

struct bpf_prog_ops {
	int (*test_run)(struct bpf_prog *prog, const union bpf_attr *kattr,
			union bpf_attr __user *uattr);
};
```


## struct bpf_prog_offload_ops

实现硬件卸载功能。

```c
struct bpf_prog_offload_ops {
	/* verifier basic callbacks */
	int (*insn_hook)(struct bpf_verifier_env *env,
			 int insn_idx, int prev_insn_idx);
	int (*finalize)(struct bpf_verifier_env *env);
	/* verifier optimization callbacks (called after .finalize) */
	int (*replace_insn)(struct bpf_verifier_env *env, u32 off,
			    struct bpf_insn *insn);
	int (*remove_insns)(struct bpf_verifier_env *env, u32 off, u32 cnt);
	/* program management callbacks */
	int (*prepare)(struct bpf_prog *prog);
	int (*translate)(struct bpf_prog *prog);
	void (*destroy)(struct bpf_prog *prog);
};
```

```

```mermaid
eBPF 程序-->load-->verifier->jit
eBPF 程序-->attach
```
截止目前eBPF支持有30余种程序类型，一部分是为了进行跟踪，如BPF_PROG_TYPE_KPROBE、BPF_PROG_TYPE_TRACEPOINT、BPF_PROG_TYPE_TRACING 、BPF_PROG_TYPE_SYSCALL等；一部分是为了高性能的网络，如BPF_PROG_TYPE_XDP、


# 跟踪、诊断、观测相关

跟踪、诊断、观测有关的eBPF程序类型主要有：

- BPF_PROG_TYPE_KPROBE
- BPF_PROG_TYPE_TRACEPOINT
- BPF_PROG_TYPE_RAW_TRACEPOINT
- BPF_PROG_TYPE_PERF_EVENT
- BPF_PROG_TYPE_TRACING：kfunc
- BPF_PROG_TYPE_SYSCALL
- BPF_PROG_TYPE_RAW_TRACEPOINT_WRITABLE



BPF_PROG_TYPE_TRACING
	BPF_MODIFY_RETURN
	BPF_TRACE_FENTRY
	BPF_TRACE_FEXIT
	BPF_TRACE_ITER
	BPF_TRACE_RAW_TP : raw tracepoint + btf，可以直接访问成员

# 高性能网络
高性能网络有关的eBPF程序类型有：
1）：socket相关：BPF_PROG_TYPE_SOCKET_FILTER、BPF_PROG_TYPE_CGROUP_SKB、BPF_PROG_TYPE_CGROUP_SOCK、BPF_PROG_TYPE_SOCK_OPS、BPF_PROG_TYPE_SK_SKB、BPF_PROG_TYPE_SK_MSG、BPF_PROG_TYPE_CGROUP_SOCK_ADDR、BPF_PROG_TYPE_SK_REUSEPORT、BPF_PROG_TYPE_CGROUP_SOCKOPT、BPF_PROG_TYPE_STRUCT_OPS、BPF_PROG_TYPE_SK_LOOKUP
BPF_PROG_TYPE_XDP

# 轻量级tunnel：
BPF_PROG_TYPE_LWT_IN、BPF_PROG_TYPE_LWT_OUT、BPF_PROG_TYPE_LWT_XMIT、BPF_PROG_TYPE_LWT_SEG6LOCAL

# Tc相关：BPF_PROG_TYPE_SCHED_CLS、BPF_PROG_TYPE_SCHED_ACT
3.2.3	安全相关
BPF_PROG_TYPE_LSM

# 3.2.4	其它的
BPF_PROG_TYPE_CGROUP_DEVICE、BPF_PROG_TYPE_LIRC_MODE2、BPF_PROG_TYPE_FLOW_DISSECTOR、BPF_PROG_TYPE_CGROUP_SYSCTL、BPF_PROG_TYPE_EXT

# 3.2.5	kprobe

# 3.2.6	tracepoint





