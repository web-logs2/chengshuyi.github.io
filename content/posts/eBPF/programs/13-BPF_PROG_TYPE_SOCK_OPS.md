---
title: "eBPF 程序类型解析系列之 BPF_PROG_TYPE_SOCK_OPS"
date: 2023-04-22T14:08:58+08:00 
description: ""
draft: false
tags: [eBPF, eBPF 程序类型系列,BPF_PROG_TYPE_SOCK_OPS]
categories: [eBPF]
---

[bpf: Adding support for sock_ops](https://lore.kernel.org/all/5956DFB1.9070005@iogearbox.net/T/#m8cb164a94ba1c242d15b96883bed8a631d51563e)


## 应用场景

socket粒度的rtt调优，BPF_SOCK_OPS_BASE_RTT

socket粒度的内存调优，BPF_SOCK_OPS_RWND_INIT
        BPF_SOCK_OPS_PASSIVE_ESTABLISHED_CB 被动建连
        BPF_SOCK_OPS_TCP_CONNECT_CB(主动建链)，通过bpf_setsockopt设置sndbuf，rcvbuf


## section

SEC("sockops")



## 入参

struct bpf_sock_ops

```c

const struct bpf_verifier_ops sock_ops_verifier_ops = {
	.get_func_proto		= sock_ops_func_proto,
	.is_valid_access	= sock_ops_is_valid_access,
	.convert_ctx_access	= sock_ops_convert_ctx_access,
};
```
## 返回值

```c
static inline int tcp_call_bpf(struct sock *sk, int op, u32 nargs, u32 *args)
{
	struct bpf_sock_ops_kern sock_ops;
	int ret;

	memset(&sock_ops, 0, offsetof(struct bpf_sock_ops_kern, temp));
	if (sk_fullsock(sk)) {
		sock_ops.is_fullsock = 1;
		sock_owned_by_me(sk);
	}

	sock_ops.sk = sk;
	sock_ops.op = op;
	if (nargs > 0)
		memcpy(sock_ops.args, args, nargs * sizeof(*args));

	ret = BPF_CGROUP_RUN_PROG_SOCK_OPS(&sock_ops);
	if (ret == 0)
		ret = sock_ops.reply;
	else
		ret = -1;
	return ret;
}

```

## attach type

BPF_CGROUP_SOCK_OPS

bpf_prog_attach(prog_fd, cg_fd, BPF_CGROUP_SOCK_OPS, 0);

    cgroup_bpf_prog_attach
        根据cg_fd找到struct cgroup，保存到cgroup结构体

## 涉及到的内核函数

## 运行
BPF_SOCK_OPS_VOID：该操作码表示BPF程序不会对套接字操作进行任何修改或操作，而是仅仅作为一个观察者。
BPF_SOCK_OPS_TIMEOUT_INIT：该操作码表示BPF程序将在套接字超时时间初始化时被调用，可以在此操作码中设置套接字的超时时间。
BPF_SOCK_OPS_RWND_INIT：该操作码表示BPF程序将在套接字缓冲区大小初始化时被调用，可以在此操作码中设置套接字的缓冲区大小。
BPF_SOCK_OPS_TCP_CONNECT_CB：该操作码表示BPF程序将在套接字连接时被调用，可以在此操作码中实现自定义逻辑。
BPF_SOCK_OPS_ACTIVE_ESTABLISHED_CB：该操作码表示BPF程序将在主动连接套接字建立连接时被调用，可以在此操作码中实现自定义逻辑。

tcp_rcv_state_process
    tcp_rcv_synsent_state_process
        tcp_finish_connect
            tcp_init_transfer
                bpf_skops_established
                    BPF_CGROUP_RUN_PROG_SOCK_OPS

BPF_SOCK_OPS_PASSIVE_ESTABLISHED_CB：该操作码表示BPF程序将在被动连接套接字建立连接时被调用，可以在此操作码中实现自定义逻辑。

处于TCP_SYN_RECV状态时，收到了第三次握手的ack包

tcp_rcv_state_process
    tcp_init_transfer

BPF_SOCK_OPS_NEEDS_ECN：该操作码表示BPF程序将在套接字需要启用显式拥塞通知时被调用，可以在此操作码中设置套接字的拥塞通知选项。
BPF_SOCK_OPS_BASE_RTT：该操作码表示BPF程序将在套接字的初始RTT计算时被调用，可以在此操作码中进行自定义逻辑。
BPF_SOCK_OPS_RTO_CB：该操作码表示BPF程序将在套接字的重传超时时被调用，可以在此操作码中进行自定义逻辑。
BPF_SOCK_OPS_RETRANS_CB：该操作码表示BPF程序将在套接字的重传时被调用，可以在此操作码中进行自定义逻辑。
BPF_SOCK_OPS_STATE_CB：该操作码表示BPF程序将在套接字状态变更时被调用，可以在此操作码中进行自定义逻辑。
BPF_SOCK_OPS_TCP_LISTEN_CB：该操作码表示BPF程序将在被动连接套接字调用listen()函数时被调用，可以在此操作码中进行自定义逻辑。
BPF_SOCK_OPS_RTT_CB：该操作码表示BPF程序将在套接字的RTT计算时被调用，可以在此操作码中进行自定义逻辑。
BPF_SOCK_OPS_PARSE_HDR_OPT_CB：该操作码表示BPF程序将在解析套接字头部选项时被调用，可以在此操作码中进行自定义逻辑。
BPF_SOCK_OPS_HDR_OPT_LEN_CB：该操作码表示BPF程序将在查询套接字头部选项长度时被调用，可以在此操作码中进行自定义逻辑。
BPF_SOCK_OPS_WRITE_HDR_OPT_CB：该操作码表示BPF程序将在写入套接字头部选项时被调用，可以在此操作码中进行自定义逻辑。






这里所说的 socket 事件包括建连（connection establishment）、重传（retransmit）、超时（timeout）等等。

场景一：动态跟踪：监听 socket 事件
这种场景只会拦截和解析系统事件，不会修改任何东西。

场景二：动态修改 socket（例如 tcp 建连）操作
拦截到事件后，通过 bpf_setsockopt() 动态修改 socket 配置， 能够实现 per-connection 的优化，提升性能。例如，

监听到被动建连（passive establishment of a connection）事件时，如果 对端和本机不在同一个网段，就动态修改这个 socket 的 MTU。 这样能避免包因为太大而被中间路由器分片（fragmenting）。
替换目的 IP 地址，实现高性能的透明代理及负载均衡。 Cilium 对 K8s 的 Service 就是这样实现的，详见 [1]。
场景三：socket redirection（需要 BPF_PROG_TYPE_SK_SKB 程序配合）
这个其实算是“动态修改”的特例。与 BPF_PROG_TYPE_SK_SKB 程序配合，通过 sockmap+redirection 实现 socket 重定向。这种情况下分为两段 BPF 程序，

第一段是 BPF_PROG_TYPE_SOCK_OPS 程序，拦截 socket 事件，并从 struct bpf_sock_ops 中提取 socket 信息存储到 sockmap；
第二段是 BPF_PROG_TYPE_SK_SKB 类型程序，从拦截到的 socket message 提取 socket 信息，然后去 sockmap 查找对端 socket，然后通过 bpf_sk_redirect_map() 直接重定向过去。
Hook 位置：多个地方
其他类型的 BPF 程序都是在某个特定的代码出执行的，但 SOCK_OPS 程序不同，它们 在多个地方执行，op 字段表示触发执行的地方。 op 字段是枚举类型，完整列表：

// include/uapi/linux/bpf.h

/* List of known BPF sock_ops operators. */
enum {
    BPF_SOCK_OPS_VOID,
    BPF_SOCK_OPS_TIMEOUT_INIT,          // 初始化 TCP RTO 时调用 BPF 程序
                                        //   程序应当返回希望使用的 SYN-RTO 值；-1 表示使用默认值
    BPF_SOCK_OPS_RWND_INIT,             // BPF 程序应当返回 initial advertized window (in packets)；-1 表示使用默认值
    BPF_SOCK_OPS_TCP_CONNECT_CB,        // 主动建连 初始化之前 回调 BPF 程序
    BPF_SOCK_OPS_ACTIVE_ESTABLISHED_CB, // 主动建连 成功之后   回调 BPF 程序
    BPF_SOCK_OPS_PASSIVE_ESTABLISHED_CB,// 被动建连 成功之后   回调 BPF 程序
    BPF_SOCK_OPS_NEEDS_ECN,             // If connection's congestion control needs ECN */
    BPF_SOCK_OPS_BASE_RTT,              // 获取 base RTT。The correct value is based on the path，可能还与拥塞控制
                                        //   算法相关。In general it indicates
                                        //   a congestion threshold. RTTs above this indicate congestion
    BPF_SOCK_OPS_RTO_CB,                // 触发 RTO（超时重传）时回调 BPF 程序，三个参数：
                                        //   Arg1: value of icsk_retransmits
                                        //   Arg2: value of icsk_rto
                                        //   Arg3: whether RTO has expired
    BPF_SOCK_OPS_RETRANS_CB,            // skb 发生重传之后，回调 BPF 程序，三个参数：
                                        //   Arg1: sequence number of 1st byte
                                        //   Arg2: # segments
                                        //   Arg3: return value of tcp_transmit_skb (0 => success)
    BPF_SOCK_OPS_STATE_CB,              // TCP 状态发生变化时，回调 BPF 程序。参数：
                                        //   Arg1: old_state
                                        //   Arg2: new_state
    BPF_SOCK_OPS_TCP_LISTEN_CB,         // 执行 listen(2) 系统调用，socket 进入 LISTEN 状态之后，回调 BPF 程序
};
从以上注释可以看到，这些 OPS 分为两种类型：

通过 BPF 程序的返回值来动态修改配置，类型包括

BPF_SOCK_OPS_TIMEOUT_INIT
BPF_SOCK_OPS_RWND_INIT
BPF_SOCK_OPS_NEEDS_ECN
BPF_SOCK_OPS_BASE_RTT
在 socket/tcp 状态发生变化时，回调（callback）BPF 程序，类型包括

BPF_SOCK_OPS_TCP_CONNECT_CB
BPF_SOCK_OPS_ACTIVE_ESTABLISHED_CB
BPF_SOCK_OPS_PASSIVE_ESTABLISHED_CB
BPF_SOCK_OPS_RTO_CB
BPF_SOCK_OPS_RETRANS_CB
BPF_SOCK_OPS_STATE_CB
BPF_SOCK_OPS_TCP_LISTEN_CB
引入该功能的内核 patch 见 bpf: Adding support for sock_ops；

SOCK_OPS 类型的 BPF 程序都是从 tcp_call_bpf() 调用过来的，这个文件中多个地方都会调用到该函数。

程序签名
传入参数： struct bpf_sock_ops *
结构体 定义，

// include/uapi/linux/bpf.h

struct bpf_sock_ops {
    __u32 op;               // socket 事件类型，就是上面的 BPF_SOCK_OPS_*
    union {
        __u32 args[4];      // Optionally passed to bpf program
        __u32 reply;        // BPF 程序的返回值。例如，op==BPF_SOCK_OPS_TIMEOUT_INIT 时，
                            //   BPF 程序的返回值就表示希望为这个 TCP 连接设置的 RTO 值
        __u32 replylong[4]; // Optionally returned by bpf prog
    };
    __u32 family;
    __u32 remote_ip4;        /* Stored in network byte order */
    __u32 local_ip4;         /* Stored in network byte order */
    __u32 remote_ip6[4];     /* Stored in network byte order */
    __u32 local_ip6[4];      /* Stored in network byte order */
    __u32 remote_port;       /* Stored in network byte order */
    __u32 local_port;        /* stored in host byte order */
    ...
};
返回值
如前面所述，ops 类型不同，返回值也不同。

加载方式：attach 到某个 cgroup（可使用 bpftool 等工具）
指定以 BPF_CGROUP_SOCK_OPS 类型，将 BPF 程序 attach 到某个 cgroup 文件描述符。

依赖 cgroupv2。

内核已经有了 BPF_PROG_TYPE_CGROUP_SOCK 类型的 BPF 程序，这里为什么又要引入一个 BPF_PROG_TYPE_SOCK_OPS 类型的程序呢？

BPF_PROG_TYPE_CGROUP_SOCK 类型的 BPF 程序：在一个连接（connection）的生命周期中只执行一次，
BPF_PROG_TYPE_SOCK_OPS 类型的 BPF 程序：在一个连接的生命周期中，在不同地方被多次调用。
程序示例
1. Customize TCP initial RTO (retransmission timeout) with BPF
2. Cracking kubernetes node proxy (aka kube-proxy)，其中的第五种实现方式
3. (译) 利用 ebpf sockmap/redirection 提升 socket 性能（2020）
延伸阅读
bpf: Adding support for sock_ops



BPF_SOCK_OPS_VOID,BPF_SOCK_OPS_TIMEOUT_INIT,BPF_SOCK_OPS_RWND_INIT,BPF_SOCK_OPS_TCP_CONNECT_CB,BPF_SOCK_OPS_ACTIVE_ESTABLISHED_CB,BPF_SOCK_OPS_PASSIVE_ESTABLISHED_CB,BPF_SOCK_OPS_NEEDS_ECN,BPF_SOCK_OPS_BASE_RTT,BPF_SOCK_OPS_RTO_CB,BPF_SOCK_OPS_RETRANS_CB,BPF_SOCK_OPS_STATE_CB,BPF_SOCK_OPS_TCP_LISTEN_CB,BPF_SOCK_OPS_RTT_CB,BPF_SOCK_OPS_PARSE_HDR_OPT_CB,BPF_SOCK_OPS_HDR_OPT_LEN_CB, BPF_SOCK_OPS_WRITE_HDR_OPT_CB,