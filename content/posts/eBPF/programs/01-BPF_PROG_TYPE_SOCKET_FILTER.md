---
title: "eBPF 程序类型解析系列之 BPF_PROG_TYPE_SOCKET_FILTER"
date: 2023-04-22T14:08:58+08:00 
description: ""
draft: true
tags: [eBPF, eBPF 程序类型系列,BPF_PROG_TYPE_SOCKET_FILTER]
categories: [eBPF]
---


[bpf: verifier: add checks for BPF_ABS | BPF_IND instructions](https://github.com/torvalds/linux/commit/ddd872bc3098f9d9abe1680a6b2013e59e3337f7)

eBPF程序类型BPF_PROG_TYPE_SOCKET_FILTER的主要功能是在Linux内核网络数据包收发路径中进行流量控制和过滤。该类型的程序可以被附加到一个socket上，用于过滤和修改通过该socket的数据包。这使得用户空间程序可以在内核中拦截和处理网络数据包，而无需将数据包传输到用户空间进行处理。这种流量控制和过滤的方式可以提高网络性能和安全，同时也可以在内核中减少不必要的数据包传输。BPF_PROG_TYPE_SOCKET_FILTER类型的程序通常用于实现网络防火墙、负载均衡、数据包重写等功能。



## 入参

convert_ctx_access 会将我们的访问转换成__sk_buff

struct __sk_buff*

gen_ld_abs 主要是用来转换BPF_LD_ABS指令

```c
const struct bpf_verifier_ops sk_filter_verifier_ops = {
	.get_func_proto		= sk_filter_func_proto,
	.is_valid_access	= sk_filter_is_valid_access,
	.convert_ctx_access	= bpf_convert_ctx_access,
	.gen_ld_abs		= bpf_gen_ld_abs,
};
```

## 返回值

return 0; 则会丢弃该报
< skb->len 则会trim
> skb->len 则没变化

```c
int sk_filter_trim_cap(struct sock *sk, struct sk_buff *skb, unsigned int cap)
{
	int err;
	struct sk_filter *filter;
    // ......
	rcu_read_lock();
	filter = rcu_dereference(sk->sk_filter);
	if (filter) {
		struct sock *save_sk = skb->sk;
		unsigned int pkt_len;

		skb->sk = sk;
        // run socket filter eBPF program
		pkt_len = bpf_prog_run_save_cb(filter->prog, skb);
		skb->sk = save_sk;
        // 
		err = pkt_len ? pskb_trim(skb, max(cap, pkt_len)) : -EPERM;
	}
	rcu_read_unlock();

	return err;
}
```

## 

## Attach type



assert(setsockopt(sock, SOL_SOCKET, SO_ATTACH_BPF, prog_fd,
                          sizeof(prog_fd[0])) == 0);


sk_setsockopt
    sk_attach_bpf
        __sk_attach_prog
            sk->sk_filter

tun_net_xmit
sk_filter_trim_cap

__sk_receive_skb
tcp_filter

tcp_v4_rcv
udp_queue_rcv_one_skb
udpv6_queue_rcv_one_skb
tcp_v6_rcv