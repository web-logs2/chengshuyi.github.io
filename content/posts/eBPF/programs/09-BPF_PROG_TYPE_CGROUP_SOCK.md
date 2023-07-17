---
title: "eBPF 程序类型解析系列之 BPF_PROG_TYPE_CGROUP_SOCK"
date: 2023-04-22T14:08:58+08:00 
description: ""
draft: false
tags: [eBPF, eBPF 程序类型系列,BPF_PROG_TYPE_CGROUP_SOCK]
categories: [eBPF]
---


[bpf: Add new cgroup attach type to enable sock modifications](https://github.com/torvalds/linux/commit/61023658760032e97869b07d54be9681d2529e77)

- BPF_CGROUP_INET4_POST_BIND cgroup/post_bind4
- BPF_CGROUP_INET6_POST_BIND cgroup/post_bind6
- BPF_CGROUP_INET_SOCK_CREATE cgroup/sock_create
- BPF_CGROUP_INET_SOCK_RELEASE cgroup/sock_release


inet_create
inet6_create