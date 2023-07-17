---
title: "eBPF 程序类型解析系列之 BPF_PROG_TYPE_PERF_EVENT"
date: 2023-04-22T14:08:58+08:00 
description: ""
draft: false
tags: [eBPF, eBPF 程序类型系列,BPF_PROG_TYPE_PERF_EVENT]
categories: [eBPF]
---



[bpf: introduce BPF_PROG_TYPE_PERF_EVENT program type](https://lore.kernel.org/lkml/1472680243-1326608-3-git-send-email-ast@fb.com/)




SEC("perf_event")


BPF_PERF_EVENT: 和 link绑定

## laod

应该还是正常的load，需要通过perf_event_verifier_ops检查即可

## attach


perf_event_open

如果支持link的话，则BPF_LINK_CREATE
否则用ioctl PERF_EVENT_IOC_SET_BPF PERF_EVENT_IOC_ENABLE

```c
	struct perf_event_attr pe_sample_attr = {
		.type = PERF_TYPE_SOFTWARE,
		.freq = 1,
		.sample_period = freq,
		.config = PERF_COUNT_SW_CPU_CLOCK,
		.inherit = 1,
	};

	for (i = 0; i < nr_cpus; i++) {
		pmu_fd[i] = perf_event_open(&pe_sample_attr, -1 /* pid */, i,
					    -1 /* group_fd */, 0 /* flags */);
		if (pmu_fd[i] < 0) {
			fprintf(stderr, "ERROR: Initializing perf sampling\n");
			return 1;
		}
		// 

	}

	return 0;
}

```


## detach

- PERF_EVENT_IOC_DISABLE
- close(pfd, fd)