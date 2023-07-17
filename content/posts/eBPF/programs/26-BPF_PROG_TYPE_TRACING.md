---
title: "eBPF 程序类型解析系列之 BPF_PROG_TYPE_TRACING"
date: 2023-04-22T14:08:58+08:00 
description: ""
draft: false
tags: [eBPF, eBPF 程序类型系列,BPF_PROG_TYPE_TRACING]
categories: [eBPF]
---




`BPF_PROG_TYPE_TRACING` 是基于btf的tracing。这里重点强调下btf-based tracing。btf-based tracing带来的最大功能就是不再需要通过bpf_probe_read来读取数据，可以直接解引用。目前BPF_PROG_TYPE_TRACING有四种不同的attach类型，分别是：

- BPF_TRACE_RAW_TP : raw tracepoint + btf，可以直接访问成员
- BPF_TRACE_FENTRY: 类似于kprobe，但是是
- BPF_TRACE_FEXIT：类似于kretprobe
- BPF_MODIFY_RETURN：
- BPF_TRACE_ITER 


## BPF_TRACE_RAW_TP

BPF_TRACE_RAW_TP是raw tracepoint + btf，能够直接访问结构体成员，无需调用bpf_probe_read。


[bpf: revolutionize bpf tracing](https://lore.kernel.org/bpf/20191016032505.2089704-1-ast@kernel.org/)



### load 流程


1. 根据跟踪的函数名字，获取其BTF编号。
2. 像一般的程序进行load就可以。


### attach分析



## BPF_TRACE_FENTRY


[Introduce BPF trampoline](https://lore.kernel.org/bpf/20191114185720.1641606-1-ast@kernel.org/)

## BPF_TRACE_ITER

BPF_TRACE_ITER 主要用于


[bpf: allow loading of a bpf_iter program](https://lore.kernel.org/bpf/20200509175900.2474947-1-yhs@fb.com/)


### link 分析


也是通过link机制，获取程序的fd，然后通过bpf_link_create来创建

```c
static int link_create(union bpf_attr *attr, bpfptr_t uattr)
{
    ......
    switch (prog->type){
    ......
    case BPF_PROG_TYPE_TRACING:
        if (prog->expected_attach_type == BPF_TRACE_RAW_TP)
			ret = bpf_raw_tp_link_attach(prog, NULL);
		else if (prog->expected_attach_type == BPF_TRACE_ITER)
			ret = bpf_iter_link_attach(attr, uattr, prog);
    ......
}

```



```c

int bpf_iter_link_attach(const union bpf_attr *attr, bpfptr_t uattr,
			 struct bpf_prog *prog)
{
	struct bpf_iter_target_info *tinfo = NULL, *iter;
	struct bpf_link_primer link_primer;
	union bpf_iter_link_info linfo;
	struct bpf_iter_link *link;
	u32 prog_btf_id, linfo_len;
	bpfptr_t ulinfo;
	int err;

    // 附载的函数
	prog_btf_id = prog->aux->attach_btf_id;
	mutex_lock(&targets_mutex);
	list_for_each_entry(iter, &targets, list) {
		if (iter->btf_id == prog_btf_id) {
			tinfo = iter;
			break;
		}
	}
	mutex_unlock(&targets_mutex);
	if (!tinfo)
		return -ENOENT;
    ......
	bpf_link_init(&link->link, BPF_LINK_TYPE_ITER, &bpf_iter_link_lops, prog);
	link->tinfo = tinfo;

	err = bpf_link_prime(&link->link, &link_primer);
	if (err) {
		kfree(link);
		return err;
	}

	if (tinfo->reg_info->attach_target) {
		err = tinfo->reg_info->attach_target(prog, &linfo, &link->aux);
		if (err) {
			bpf_link_cleanup(&link_primer);
			return err;
		}
	}

	return bpf_link_settle(&link_primer);
}
```

### 查看内核有哪些iter


`bpftrace -l | grep iter:`