---
title: "eBPF 程序类型解析系列之 BPF_PROG_TYPE_EXT"
date: 2023-04-22T14:08:58+08:00 
description: ""
draft: false
tags: [eBPF, eBPF 程序类型系列,BPF_PROG_TYPE_EXT]
categories: [eBPF]
---


[bpf: Introduce dynamic program extensions](https://lore.kernel.org/bpf/20200121005348.2769920-2-ast@kernel.org/)

用于程序替换，可以替换掉某个eBPF程序内的子函数


```c
// SPDX-License-Identifier: GPL-2.0
/* Copyright (c) 2019 Facebook */
#include <linux/stddef.h>
#include <linux/if_ether.h>
#include <linux/ipv6.h>
#include <linux/bpf.h>
#include <linux/tcp.h>
#include <bpf/bpf_helpers.h>
#include <bpf/bpf_endian.h>
#include <bpf/bpf_tracing.h>

struct sk_buff {
	unsigned int len;
};

__u64 test_result = 0;
SEC("fexit/test_pkt_access")
int BPF_PROG(test_main, struct sk_buff *skb, int ret)
{
	int len;

	__builtin_preserve_access_index(({
		len = skb->len;
	}));
	if (len != 74 || ret != 0)
		return 0;
	test_result = 1;
	return 0;
}

__u64 test_result_subprog1 = 0;
SEC("fexit/test_pkt_access_subprog1")
int BPF_PROG(test_subprog1, struct sk_buff *skb, int ret)
{
	int len;

	__builtin_preserve_access_index(({
		len = skb->len;
	}));
	if (len != 74 || ret != 148)
		return 0;
	test_result_subprog1 = 1;
	return 0;
}

/* Though test_pkt_access_subprog2() is defined in C as:
 * static __attribute__ ((noinline))
 * int test_pkt_access_subprog2(int val, volatile struct __sk_buff *skb)
 * {
 *     return skb->len * val;
 * }
 * llvm optimizations remove 'int val' argument and generate BPF assembly:
 *   r0 = *(u32 *)(r1 + 0)
 *   w0 <<= 1
 *   exit
 * In such case the verifier falls back to conservative and
 * tracing program can access arguments and return value as u64
 * instead of accurate types.
 */
struct args_subprog2 {
	__u64 args[5];
	__u64 ret;
};
__u64 test_result_subprog2 = 0;
SEC("fexit/test_pkt_access_subprog2")
int test_subprog2(struct args_subprog2 *ctx)
{
	struct sk_buff *skb = (void *)ctx->args[0];
	__u64 ret;
	int len;

	bpf_probe_read_kernel(&len, sizeof(len),
			      __builtin_preserve_access_index(&skb->len));

	ret = ctx->ret;
	/* bpf_prog_test_load() loads "test_pkt_access.bpf.o" with
	 * BPF_F_TEST_RND_HI32 which randomizes upper 32 bits after BPF_ALU32
	 * insns. Hence after 'w0 <<= 1' upper bits of $rax are random. That is
	 * expected and correct. Trim them.
	 */
	ret = (__u32) ret;
	if (len != 74 || ret != 148)
		return 0;
	test_result_subprog2 = 1;
	return 0;
}

__u64 test_result_subprog3 = 0;
SEC("fexit/test_pkt_access_subprog3")
int BPF_PROG(test_subprog3, int val, struct sk_buff *skb, int ret)
{
	int len;

	__builtin_preserve_access_index(({
		len = skb->len;
	}));
	if (len != 74 || ret != 74 * val || val != 3)
		return 0;
	test_result_subprog3 = 1;
	return 0;
}

__u64 test_get_skb_len = 0;
SEC("freplace/get_skb_len")
int new_get_skb_len(struct __sk_buff *skb)
{
	int len = skb->len;

	if (len != 74)
		return 0;
	test_get_skb_len = 1;
	return 74; /* original get_skb_len() returns skb->len */
}

__u64 test_get_skb_ifindex = 0;
SEC("freplace/get_skb_ifindex")
int new_get_skb_ifindex(int val, struct __sk_buff *skb, int var)
{
	void *data_end = (void *)(long)skb->data_end;
	void *data = (void *)(long)skb->data;
	struct ipv6hdr ip6, *ip6p;
	int ifindex = skb->ifindex;

	/* check that BPF extension can read packet via direct packet access */
	if (data + 14 + sizeof(ip6) > data_end)
		return 0;
	ip6p = data + 14;

	if (ip6p->nexthdr != 6 || ip6p->payload_len != __bpf_constant_htons(123))
		return 0;

	/* check that legacy packet access helper works too */
	if (bpf_skb_load_bytes(skb, 14, &ip6, sizeof(ip6)) < 0)
		return 0;
	ip6p = &ip6;
	if (ip6p->nexthdr != 6 || ip6p->payload_len != __bpf_constant_htons(123))
		return 0;

	if (ifindex != 1 || val != 3 || var != 1)
		return 0;
	test_get_skb_ifindex = 1;
	return 3; /* original get_skb_ifindex() returns val * ifindex * var */
}

volatile __u64 test_get_constant = 0;
SEC("freplace/get_constant")
int new_get_constant(long val)
{
	if (val != 123)
		return 0;
	test_get_constant = 1;
	return test_get_constant; /* original get_constant() returns val - 122 */
}

__u64 test_pkt_write_access_subprog = 0;
SEC("freplace/test_pkt_write_access_subprog")
int new_test_pkt_write_access_subprog(struct __sk_buff *skb, __u32 off)
{

	void *data = (void *)(long)skb->data;
	void *data_end = (void *)(long)skb->data_end;
	struct tcphdr *tcp;

	if (off > sizeof(struct ethhdr) + sizeof(struct ipv6hdr))
		return -1;

	tcp = data + off;
	if (tcp + 1 > data_end)
		return -1;

	/* make modifications to the packet data */
	tcp->check++;
	tcp->syn = 0;

	test_pkt_write_access_subprog = 1;
	return 0;
}

char _license[] SEC("license") = "GPL";
```


```c
// SPDX-License-Identifier: GPL-2.0-only
/* Copyright (c) 2017 Facebook
 */
#include <stddef.h>
#include <string.h>
#include <linux/bpf.h>
#include <linux/if_ether.h>
#include <linux/if_packet.h>
#include <linux/ip.h>
#include <linux/ipv6.h>
#include <linux/in.h>
#include <linux/tcp.h>
#include <linux/pkt_cls.h>
#include <bpf/bpf_helpers.h>
#include <bpf/bpf_endian.h>
#include "bpf_misc.h"

/* llvm will optimize both subprograms into exactly the same BPF assembly
 *
 * Disassembly of section .text:
 *
 * 0000000000000000 test_pkt_access_subprog1:
 * ; 	return skb->len * 2;
 *        0:	61 10 00 00 00 00 00 00	r0 = *(u32 *)(r1 + 0)
 *        1:	64 00 00 00 01 00 00 00	w0 <<= 1
 *        2:	95 00 00 00 00 00 00 00	exit
 *
 * 0000000000000018 test_pkt_access_subprog2:
 * ; 	return skb->len * val;
 *        3:	61 10 00 00 00 00 00 00	r0 = *(u32 *)(r1 + 0)
 *        4:	64 00 00 00 01 00 00 00	w0 <<= 1
 *        5:	95 00 00 00 00 00 00 00	exit
 *
 * Which makes it an interesting test for BTF-enabled verifier.
 */
static __attribute__ ((noinline))
int test_pkt_access_subprog1(volatile struct __sk_buff *skb)
{
	return skb->len * 2;
}

static __attribute__ ((noinline))
int test_pkt_access_subprog2(int val, volatile struct __sk_buff *skb)
{
	return skb->len * val;
}

#define MAX_STACK (512 - 2 * 32)

__attribute__ ((noinline))
int get_skb_len(struct __sk_buff *skb)
{
	volatile char buf[MAX_STACK] = {};

	__sink(buf[MAX_STACK - 1]);

	return skb->len;
}

__attribute__ ((noinline))
int get_constant(long val)
{
	return val - 122;
}

int get_skb_ifindex(int, struct __sk_buff *skb, int);

__attribute__ ((noinline))
int test_pkt_access_subprog3(int val, struct __sk_buff *skb)
{
	return get_skb_len(skb) * get_skb_ifindex(val, skb, get_constant(123));
}

__attribute__ ((noinline))
int get_skb_ifindex(int val, struct __sk_buff *skb, int var)
{
	volatile char buf[MAX_STACK] = {};

	__sink(buf[MAX_STACK - 1]);

	return skb->ifindex * val * var;
}

__attribute__ ((noinline))
int test_pkt_write_access_subprog(struct __sk_buff *skb, __u32 off)
{
	void *data = (void *)(long)skb->data;
	void *data_end = (void *)(long)skb->data_end;
	struct tcphdr *tcp = NULL;

	if (off > sizeof(struct ethhdr) + sizeof(struct ipv6hdr))
		return -1;

	tcp = data + off;
	if (tcp + 1 > data_end)
		return -1;
	/* make modification to the packet data */
	tcp->check++;
	return 0;
}

SEC("tc")
int test_pkt_access(struct __sk_buff *skb)
{
	void *data_end = (void *)(long)skb->data_end;
	void *data = (void *)(long)skb->data;
	struct ethhdr *eth = (struct ethhdr *)(data);
	struct tcphdr *tcp = NULL;
	__u8 proto = 255;
	__u64 ihl_len;

	if (eth + 1 > data_end)
		return TC_ACT_SHOT;

	if (eth->h_proto == bpf_htons(ETH_P_IP)) {
		struct iphdr *iph = (struct iphdr *)(eth + 1);

		if (iph + 1 > data_end)
			return TC_ACT_SHOT;
		ihl_len = iph->ihl * 4;
		proto = iph->protocol;
		tcp = (struct tcphdr *)((void *)(iph) + ihl_len);
	} else if (eth->h_proto == bpf_htons(ETH_P_IPV6)) {
		struct ipv6hdr *ip6h = (struct ipv6hdr *)(eth + 1);

		if (ip6h + 1 > data_end)
			return TC_ACT_SHOT;
		ihl_len = sizeof(*ip6h);
		proto = ip6h->nexthdr;
		tcp = (struct tcphdr *)((void *)(ip6h) + ihl_len);
	}

	if (test_pkt_access_subprog1(skb) != skb->len * 2)
		return TC_ACT_SHOT;
	if (test_pkt_access_subprog2(2, skb) != skb->len * 2)
		return TC_ACT_SHOT;
	if (test_pkt_access_subprog3(3, skb) != skb->len * 3 * skb->ifindex)
		return TC_ACT_SHOT;
	if (tcp) {
		if (test_pkt_write_access_subprog(skb, (void *)tcp - data))
			return TC_ACT_SHOT;
		if (((void *)(tcp) + 20) > data_end || proto != 6)
			return TC_ACT_SHOT;
		barrier(); /* to force ordering of checks */
		if (((void *)(tcp) + 18) > data_end)
			return TC_ACT_SHOT;
		if (tcp->urg_ptr == 123)
			return TC_ACT_OK;
	}

	return TC_ACT_UNSPEC;
}

```