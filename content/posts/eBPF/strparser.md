


https://switch-router.gitee.io/blog/strparser/


```c

struct strp_callbacks {
	int (*parse_msg)(struct strparser *strp, struct sk_buff *skb);
	void (*rcv_msg)(struct strparser *strp, struct sk_buff *skb); 
	int (*read_sock_done)(struct strparser *strp, int err)
	void (*abort_parser)(struct strparser *strp, int err);
	void (*lock)(struct strparser *strp);
	void (*unlock)(struct strparser *strp);
};
```


### strpaser 截获报文

strp_data_ready
  |- strp_read_sock
    |- tcp_read_sock
       |- strp_recv
         |- __strp_recv
           |- strp->cb.parse_msg(strp, head)
           ...
           |- strp->cb.rcv_msg(strp, head);