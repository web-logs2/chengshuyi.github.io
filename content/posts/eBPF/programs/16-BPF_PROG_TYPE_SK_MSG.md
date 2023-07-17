---
title: "eBPF 程序类型解析系列之 BPF_PROG_TYPE_SK_MSG"
date: 2023-04-22T14:08:58+08:00 
description: ""
draft: false
tags: [eBPF, eBPF 程序类型系列,BPF_PROG_TYPE_SK_MSG]
categories: [eBPF]
---



[bpf: create tcp_bpf_ulp allowing BPF to monitor socket TX/RX data](https://github.com/torvalds/linux/commit/4f738adba30a7cfc006f605707e7aee847ffefa0)

https://cyral.com/blog/how-to-ebpf-accelerating-cloud-native/


## attach 


BPF_SK_MSG_VERDICT


yer (TCP_ULP_BPF) to allow BPF prgrams to be run on sendmsg/sendfile system calls and inspect data.



bpf_prog_attach(msg_prog, map_fd_msg, BPF_SK_MSG_VERDICT, 0);

int bpf_prog_attach(int prog_fd, int target_fd, enum bpf_attach_type type,
		    unsigned int flags)

---
kernel:


bpf_prog_attach
    sock_map_get_from_fd
        sock_map_prog_update


```c

static int sock_map_prog_update(struct bpf_map *map, struct bpf_prog *prog,
				struct bpf_prog *old, u32 which)
{
	struct bpf_prog **pprog;
	int ret;

    // 判断sockmap是否支持which类型，这里的which是BPF_SK_MSG_VERDICT
	ret = sock_map_prog_lookup(map, &pprog, which);
	if (ret)
		return ret;

	if (old)
		return psock_replace_prog(pprog, prog, old);

    // old 为NULL，所以这里是直接替换，将prog插入sockmap
	psock_set_prog(pprog, prog);
	return 0;
}
```

## section

`SEC("sk_msg")`

## 入参

`struct sk_msg_md *`

## 返回值
 SK_PASS and
SK_DROP. Returning SK_DROP free's the copied data in the sendmsg
case and in the sendpage case leaves the data untouched. Both cases
return -EACESS to the user. Returning SK_PASS will allow the msg to
be sent.



## 运行流程


```c

static void tcp_bpf_rebuild_protos(struct proto prot[TCP_BPF_NUM_CFGS],
				   struct proto *base)
{
	// ......
	prot[TCP_BPF_TX].sendmsg		= tcp_bpf_sendmsg;
	prot[TCP_BPF_TX].sendpage		= tcp_bpf_sendpage;
	// ......
}

static int __init tcp_bpf_v4_build_proto(void)
{
	tcp_bpf_rebuild_protos(tcp_bpf_prots[TCP_BPF_IPV4], &tcp_prot);
	return 0;
}
late_initcall(tcp_bpf_v4_build_proto);
```

tcp_bpf_sendmsg
tcp_bpf_sendpage

	tcp_bpf_send_verdict
		sk_psock_msg_verdict