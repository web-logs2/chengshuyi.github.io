---
title: "eBPF 程序类型解析系列之 BPF_PROG_TYPE_SK_SKB"
date: 2023-04-22T14:08:58+08:00 
description: ""
draft: false
tags: [eBPF, eBPF 程序类型系列,BPF_PROG_TYPE_SK_SKB]
categories: [eBPF]
---




[bpf: introduce new program type for skbs on sockets](https://lore.kernel.org/all/20170816.142244.1168135640562628662.davem@davemloft.net/T/#m72679b1b50cb2a07b17347e836ecdca44e783706)

https://www.51cto.com/article/645284.html


## strparser

strparser是 Linux 内核在 4.9 版本引入的 feature (https://lwn.net/Articles/703116/)。它允许用户在内核层面拦截送往 TCP 套接字的报文并做自定义的处理。处理的地方可以是内核模块，也可以是 eBPF 程序。

了解sk_skb，需要先了解Strparser，因为其是基于strparser框架实现的。strparser的主要工作原理是当内核收到网络包时，会通过sk_data_ready来通知用户态进程，而strparser则用strp_data_ready替换掉sk_data_ready，获得掌控权。strp_data_ready则会在内部调用parse_msg和rcv_msg，parse_msg则是对报文数据处理，返回应接受的报文长度，rcv_msg则决定了如何接收报文。
```
tcp_rcv_established
    tcp_data_queue
        strp_data_ready
            strp_read_sock
                tcp_read_sock
                    strp_recv
                        __strp_recv
                            strp->cb.parse_msg(strp, head)
                            strp->cb.rcv_msg(strp, head);
```

## attach type


tcp_rcv_established
tcp_rcv_state_process
    tcp_data_snd_check
        tcp_check_space
            tcp_new_space

```c
static void tcp_new_space(struct sock *sk)
{
	struct tcp_sock *tp = tcp_sk(sk);

	if (tcp_should_expand_sndbuf(sk)) {
		tcp_sndbuf_expand(sk);
		tp->snd_cwnd_stamp = tcp_jiffies32;
	}

	INDIRECT_CALL_1(sk->sk_write_space, sk_stream_write_space, sk);
}
```

### BPF_SK_SKB_STREAM_PARSER

SEC("sk_skb/stream_parser")

https://github.com/torvalds/linux/commit/464bc0fd6273d518aee79fbd37211dd9bc35d863
### BPF_SK_SKB_STREAM_VERDICT

SEC("sk_skb/stream_verdict")

[bpf: convert sockmap field attach_bpf_fd2 to type](https://github.com/torvalds/linux/commit/464bc0fd6273d518aee79fbd37211dd9bc35d863)

### BPF_SK_SKB_VERDICT

skb_verdict
- BPF_SK_SKB_VERDICT https://lore.kernel.org/bpf/20210331023237.41094-10-xiyou.wangcong@gmail.com/

## attach 流程

bpf_prog_attach
    sock_map_get_from_fd
        sock_map_prog_update
            sock_map_prog_lookup


```c
int sk_psock_init_strp(struct sock *sk, struct sk_psock *psock)
{
	static const struct strp_callbacks cb = {
		.rcv_msg	= sk_psock_strp_read,
		.read_sock_done	= sk_psock_strp_read_done,
		.parse_msg	= sk_psock_strp_parse,
	};

	return strp_init(&psock->strp, sk, &cb);
}

void sk_psock_start_strp(struct sock *sk, struct sk_psock *psock)
{
	if (psock->saved_data_ready)
		return;

	psock->saved_data_ready = sk->sk_data_ready;
	sk->sk_data_ready = sk_psock_strp_data_ready;
	sk->sk_write_space = sk_psock_write_space;
}

static void sk_psock_write_space(struct sock *sk)
{
	struct sk_psock *psock;
	void (*write_space)(struct sock *sk) = NULL;

	rcu_read_lock();
	psock = sk_psock(sk);
	if (likely(psock)) {
		if (sk_psock_test_state(psock, SK_PSOCK_TX_ENABLED))
			schedule_work(&psock->work);
		write_space = psock->saved_write_space;
	}
	rcu_read_unlock();
	if (write_space)
		write_space(sk);
}


sk_psock_backlog
```
strp_data_ready
  |- strp_read_sock
    |- tcp_read_sock
       |- strp_recv
         |- __strp_recv
           |- strp->cb.parse_msg(strp, head)
           ...
           |- strp->cb.rcv_msg(strp, head);


tcp_rcv_established
    tcp_data_queue
        sk_psock_strp_data_ready
            strp_data_ready
                strp_read_sock
                    tcp_read_sock
                        strp_recv
                            __strp_recv


```c
static int __strp_recv(read_descriptor_t *desc, struct sk_buff *orig_skb,
		       unsigned int orig_offset, size_t orig_len,
		       size_t max_msg_size, long timeo)
{

	while (eaten < orig_len) {
		/* Always clone since we will consume something */
		skb = skb_clone(orig_skb, GFP_ATOMIC);
		if (!skb) {
			STRP_STATS_INCR(strp->stats.mem_fail);
			desc->error = -ENOMEM;
			break;
		}

		cand_len = orig_len - eaten;

		head = strp->skb_head;
		if (!head) {
			head = skb;
			strp->skb_head = head;
			/* Will set skb_nextp on next packet if needed */
			strp->skb_nextp = NULL;
			stm = _strp_msg(head);
			memset(stm, 0, sizeof(*stm));
			stm->strp.offset = orig_offset + eaten;
		} else {
			/* Unclone if we are appending to an skb that we
			 * already share a frag_list with.
			 */
			if (skb_has_frag_list(skb)) {
				err = skb_unclone(skb, GFP_ATOMIC);
				if (err) {
					STRP_STATS_INCR(strp->stats.mem_fail);
					desc->error = err;
					break;
				}
			}

			stm = _strp_msg(head);
			*strp->skb_nextp = skb;
			strp->skb_nextp = &skb->next;
			head->data_len += skb->len;
			head->len += skb->len;
			head->truesize += skb->truesize;
		}

		if (!stm->strp.full_len) {
			ssize_t len;

			len = (*strp->cb.parse_msg)(strp, head);

			if (!len) {
				/* Need more header to determine length */
				if (!stm->accum_len) {
					/* Start RX timer for new message */
					strp_start_timer(strp, timeo);
				}
				stm->accum_len += cand_len;
				eaten += cand_len;
				STRP_STATS_INCR(strp->stats.need_more_hdr);
				WARN_ON(eaten != orig_len);
				break;
			} else if (len < 0) {
				if (len == -ESTRPIPE && stm->accum_len) {
					len = -ENODATA;
					strp->unrecov_intr = 1;
				} else {
					strp->interrupted = 1;
				}
				strp_parser_err(strp, len, desc);
				break;
			} else if (len > max_msg_size) {
				/* Message length exceeds maximum allowed */
				STRP_STATS_INCR(strp->stats.msg_too_big);
				strp_parser_err(strp, -EMSGSIZE, desc);
				break;
			} else if (len <= (ssize_t)head->len -
					  skb->len - stm->strp.offset) {
				/* Length must be into new skb (and also
				 * greater than zero)
				 */
				STRP_STATS_INCR(strp->stats.bad_hdr_len);
				strp_parser_err(strp, -EPROTO, desc);
				break;
			}

			stm->strp.full_len = len;
		}

		extra = (ssize_t)(stm->accum_len + cand_len) -
			stm->strp.full_len;

		if (extra < 0) {
			/* Message not complete yet. */
			if (stm->strp.full_len - stm->accum_len >
			    strp_peek_len(strp)) {
				/* Don't have the whole message in the socket
				 * buffer. Set strp->need_bytes to wait for
				 * the rest of the message. Also, set "early
				 * eaten" since we've already buffered the skb
				 * but don't consume yet per strp_read_sock.
				 */

				if (!stm->accum_len) {
					/* Start RX timer for new message */
					strp_start_timer(strp, timeo);
				}

				stm->accum_len += cand_len;
				eaten += cand_len;
				strp->need_bytes = stm->strp.full_len -
						       stm->accum_len;
				STRP_STATS_ADD(strp->stats.bytes, cand_len);
				desc->count = 0; /* Stop reading socket */
				break;
			}
			stm->accum_len += cand_len;
			eaten += cand_len;
			WARN_ON(eaten != orig_len);
			break;
		}

		/* Positive extra indicates more bytes than needed for the
		 * message
		 */

		WARN_ON(extra > cand_len);

		eaten += (cand_len - extra);

		/* Hurray, we have a new message! */
		cancel_delayed_work(&strp->msg_timer_work);
		strp->skb_head = NULL;
		strp->need_bytes = 0;
		STRP_STATS_INCR(strp->stats.msgs);

		/* Give skb to upper layer */
		strp->cb.rcv_msg(strp, head);

		// ......
	}

	return eaten;
}

static void sk_psock_strp_read(struct strparser *strp, struct sk_buff *skb)
{
	struct sk_psock *psock;
	struct bpf_prog *prog;
	int ret = __SK_DROP;
	struct sock *sk;

	rcu_read_lock();
	sk = strp->sk;
	psock = sk_psock(sk);
	if (unlikely(!psock)) {
		sock_drop(sk, skb);
		goto out;
	}
	prog = READ_ONCE(psock->progs.stream_verdict);
	if (likely(prog)) {
		skb->sk = sk;
		skb_dst_drop(skb);
		skb_bpf_redirect_clear(skb);
		ret = bpf_prog_run_pin_on_cpu(prog, skb);
		if (ret == SK_PASS)
			skb_bpf_set_strparser(skb);
		ret = sk_psock_map_verd(ret, skb_bpf_redirect_fetch(skb));
		skb->sk = NULL;
	}
	sk_psock_verdict_apply(psock, skb, ret);
out:
	rcu_read_unlock();
}


```

## 入参

struct __sk_buff *



场景一：修改 skb/socket 信息，socket 重定向
这个功能依赖 sockmap，后者是一种特殊类型的 BPF map，其中存储的是 socket 引用（references）。

典型流程：

创建 sockmap
拦截 socket 操作，将 socket 信息存入 sockmap
拦截 socket sendmsg/recvmsg 等系统调用，从 msg 中提取信息（IP、port 等），然后 在 sockmap 中查找对端 socket，然后重定向过去。
根据提取到的 socket 信息判断接下来应该做什么的过程称为 verdict（判决）。 verdict 类型可以是：

__SK_DROP
__SK_PASS
__SK_REDIRECT
场景二：动态解析消息流（stream parsing）
这种程序的一个应用是 strparser framework。

它与上层应用配合，在内核中提供应用层消息解析的支持（provide kernel support for application layer messages）。两个使用了 strparser 框架的例子：TLS 和 KCM（ Kernel Connection Multiplexor）。


