




https://lore.kernel.org/all/20170816.142244.1168135640562628662.davem@davemloft.net/T/#m87688091554dc1b9cd4e3f8f72b6c396d4206e06


sock map的key u32（目前来看是索引), value是 socket fd.

```c

const struct bpf_map_ops sock_map_ops = {
	.map_meta_equal		= bpf_map_meta_equal,
	.map_alloc		= sock_map_alloc,
	.map_free		= sock_map_free,
	.map_get_next_key	= sock_map_get_next_key,
	.map_lookup_elem_sys_only = sock_map_lookup_sys,
	.map_update_elem	= sock_map_update_elem,
	.map_delete_elem	= sock_map_delete_elem,
	.map_lookup_elem	= sock_map_lookup,
	.map_release_uref	= sock_map_release_progs,
	.map_check_btf		= map_check_no_btf,
	.map_mem_usage		= sock_map_mem_usage,
	.map_btf_id		= &sock_map_btf_ids[0],
	.iter_seq_info		= &sock_map_iter_seq_info,
};
```






## sockmap 注册strparser


bpf_sock_map_update
	sock_map_update_common
		sock_map_add_link
			sock_map_link
				sk_psock_init_strp

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
```


```c

void sk_psock_start_verdict(struct sock *sk, struct sk_psock *psock)
{
	if (psock->saved_data_ready)
		return;
	// 保存原来的sk_data_ready
	psock->saved_data_ready = sk->sk_data_ready;
	// 收的方向
	sk->sk_data_ready = sk_psock_verdict_data_ready;
	// 发的方向
	sk->sk_write_space = sk_psock_write_space;
}

```