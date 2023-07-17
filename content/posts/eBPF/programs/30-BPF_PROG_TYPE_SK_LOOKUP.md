---
title: "eBPF 程序类型解析系列之 BPF_PROG_TYPE_SK_LOOKUP"
date: 2023-04-22T14:08:58+08:00 
description: ""
draft: false
tags: [eBPF, eBPF 程序类型系列,BPF_PROG_TYPE_SK_LOOKUP]
categories: [eBPF]
---


看起来只是做skb的路由，比如tcp只需监听5201端口，通过sk_lookup，可以把端口为x的skb转发给改tcp

https://lore.kernel.org/bpf/20210514003623.28033-1-alexei.starovoitov@gmail.com/


https://github.com/jsitnicki/ebpf-summit-2020


BPF_SK_LOOKUP

- __inet_lookup_listener
- __udp4_lib_lookup
- inet6_lookup_listener
- __udp6_lib_lookup

```c
struct sock *__inet_lookup_listener(struct net *net,
				    struct inet_hashinfo *hashinfo,
				    struct sk_buff *skb, int doff,
				    const __be32 saddr, __be16 sport,
				    const __be32 daddr, const unsigned short hnum,
				    const int dif, const int sdif)
{
	struct inet_listen_hashbucket *ilb2;
	struct sock *result = NULL;
	unsigned int hash2;

	/* Lookup redirect from BPF */
	if (static_branch_unlikely(&bpf_sk_lookup_enabled)) {
		result = inet_lookup_run_bpf(net, hashinfo, skb, doff,
					     saddr, sport, daddr, hnum, dif);
		if (result)
			goto done;
	}

	hash2 = ipv4_portaddr_hash(net, daddr, hnum);
	ilb2 = inet_lhash2_bucket(hashinfo, hash2);

	result = inet_lhash2_lookup(net, ilb2, skb, doff,
				    saddr, sport, daddr, hnum,
				    dif, sdif);
	if (result)
		goto done;

	/* Lookup lhash2 with INADDR_ANY */
	hash2 = ipv4_portaddr_hash(net, htonl(INADDR_ANY), hnum);
	ilb2 = inet_lhash2_bucket(hashinfo, hash2);

	result = inet_lhash2_lookup(net, ilb2, skb, doff,
				    saddr, sport, htonl(INADDR_ANY), hnum,
				    dif, sdif);
done:
	if (IS_ERR(result))
		return NULL;
	return result;
}
```