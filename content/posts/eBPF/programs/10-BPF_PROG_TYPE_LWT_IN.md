---
title: "eBPF 程序类型解析系列之 BPF_PROG_TYPE_LWT_IN"
date: 2023-04-22T14:08:58+08:00 
description: ""
draft: false
tags: [eBPF, eBPF 程序类型系列,BPF_PROG_TYPE_LWT_IN]
categories: [eBPF]
---


[bpf: BPF for lightweight tunnel encapsulation](https://lore.kernel.org/all/1e9928e5798fed084c4cbf26fc7e9abb251a4403.1480424542.git.tgraf@suug.ch/t/#m599006b74c18b8c38e43e0567c424bd7345d6a56)


- BPF_PROG_TYPE_LWT_IN   => dst_input()
- BPF_PROG_TYPE_LWT_OUT  => dst_output()
- BPF_PROG_TYPE_LWT_XMIT => lwtunnel_xmit()


