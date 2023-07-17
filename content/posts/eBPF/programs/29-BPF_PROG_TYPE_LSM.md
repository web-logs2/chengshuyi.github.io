---
title: "eBPF 程序类型解析系列之 BPF_PROG_TYPE_LSM"
date: 2023-04-22T14:08:58+08:00 
description: ""
draft: false
tags: [eBPF, eBPF 程序类型系列,BPF_PROG_TYPE_LSM]
categories: [eBPF]
---


Introduce types and configs for bpf programs that can be attached to
LSM hooks. The programs can be enabled by the config option
CONFIG_BPF_LSM.

主要是能够加eBPF程序附载到LSM hooks。

attach type 是
- BPF_LSM_MAC     SEC("lsm/
- BPF_LSM_CGROUP： [bpf: per-cgroup lsm flavor](https://lore.kernel.org/all/20220628174314.1216643-4-sdf@google.com/) SEC("lsm_cgroup/

附载函数列表：

[bpf: Introduce BPF_PROG_TYPE_LSM](https://lore.kernel.org/bpf/20200329004356.27286-2-kpsingh@chromium.org/)
这个只引入了BPF_LSM_MAC




```c
// 导入可以attach的函数
#define LSM_HOOK(RET, DEFAULT, NAME, ...)	\
noinline RET bpf_lsm_##NAME(__VA_ARGS__)	\
{						\
	return DEFAULT;				\
}

#include <linux/lsm_hook_defs.h>
#undef LSM_HOOK

// 定义attach函数的id，避免有冲突，通过btf id set来管理
#define LSM_HOOK(RET, DEFAULT, NAME, ...) BTF_ID(func, bpf_lsm_##NAME)
BTF_SET_START(bpf_lsm_hooks)
#include <linux/lsm_hook_defs.h>
#undef LSM_HOOK
BTF_SET_END(bpf_lsm_hooks)
```