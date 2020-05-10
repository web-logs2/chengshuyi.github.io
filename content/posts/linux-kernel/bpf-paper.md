---
title: "The BSD Packet Filter论文笔记及源码解析"
date: 2020-05-09T08:36:36+08:00
description: ""
draft: false
tags: [linux内核,bpf]
categories: [linux内核]
---

本文主要介绍了bpf论文和bpf源码。

### 简介

从下图可以看出，当网卡驱动接收到packet的时，一般情况下会直接传给协议栈。但是当bpf处于工作状态，驱动首先调用bpf程序，bpf程序会执行与其绑定的filter程序。如果该packet通过过滤条件，将该packet拷贝到对应进程的buffer中。由此，bpf程序主要由两部分构成：

1. network tap：将网卡驱动收到的packet传给监听的应用程序；
2. filter：过滤掉不需要的packet。

![](https://gitee.com/chengshuyi/scripts/raw/master/img/20200509083958.png)

### network tap

网卡驱动一旦收到/发出packet，会先调用`bpf_tap`函数，下面是`bpf_tap`函数的源码。

```c
void bpf_tap(arg, pkt, pktlen)
	caddr_t arg;
	register u_char *pkt;
	register u_int pktlen;
{
	struct bpf_if *bp;
	register struct bpf_d *d;
	register u_int slen;
	// arg是和特定网卡绑定的bpf_if，类似于net_if
	bp = (struct bpf_if *)arg;
	// 遍历所有的监听描述符，该描述符对应着一个进程
	for (d = bp->bif_dlist; d != 0; d = d->bd_next) {
		++d->bd_rcount;
		// 执行每个描述符的过滤程序
		slen = bpf_filter(d->bd_filter, pkt, pktlen, pktlen);
		if (slen != 0)
			// 返回非0，将包传给该描述符，也就是拷贝到该描述符指定的buffer中；
            // 1. bpf可以设置immediate参数，每次都唤醒该描述符对应的进程
            // 2. 或者等缓冲区满的时候再唤醒该描述符对应的进程，该进程从缓冲区读取数据
			catchpacket(d, pkt, pktlen, slen, bcopy);
	}
}
```

### filter

filter是bpf实现比较关键的部分，作者首先提出了布尔表达式树和控制流图的方式来计算是否需要该packet，并指出在大部分情况下，控制流图的方式所需要的操作要少一些。

其次，作者自定义了一些伪机器码，该伪机器码同一般的机器码类似，但是不能够直接运行在真实的机器上，需要进行动态的翻译。

filter程序对应的机器模型由两个寄存器，分别是accumulator（记作A）和index register（记作x），下面列举了filter程序的指令格式、指令集及取址模式：

* 指令格式：16bit的操作码，jt是jump true，jf是jump false。

![](https://gitee.com/chengshuyi/scripts/raw/master/img/20200509095900.png)

* 指令集：
  * ld类型指令：`ld,ldh,ldb`拷贝数值到寄存器`A`，`ldx`拷贝数值到寄存器`X`；下面是执行ld类型指令的流程；
  * alu类型指令（`add,sub...`）：使用寄存器`A`、立即数k、和寄存器`x`做运算，并将运算结果存储到寄存器`A`；
  * jump类型指令：比较寄存器`A`和操作数的值，根据是否相同做跳转；
  * 存储类型指令：`st`拷贝数值到寄存器`A`，`stx`拷贝数值到寄存器`X`；
  * ret指令：表示结束，以及需要接收字节；
  * tax和txa指令：将寄存器值传递给另外一个寄存器；

![](https://gitee.com/chengshuyi/scripts/raw/master/img/20200509095351.png)

* 取址模式
  * 带`#`的表示数值在指令格式的k字段上；
  * M[k]：表示分配的临时内存中第k个字节；
  * [k]：packet的偏移地址为k上的数据；
  * L：当前指令所在的偏移地址；

![](https://gitee.com/chengshuyi/scripts/raw/master/img/20200509100321.png)

实例分析：

```assembly
	ldh [12]					# 加载packet偏移地址为12的半字到寄存器A，根据以太网包头部格式知道该半字表示以太网包中的数据类型
	jeq #ETHERPROTO_IP, L1, L5	# 比较A和ETHERPROTO_IP的值，相等去L1，不等去L5
L1: ldb [23]					# 加载packet偏移地址为23的字节到寄存器A
	jeq #IPPROTO_TCP, L2, L5	# 比较A和IPPROTO_TCP的值，相等去L2，不等去L5
L2: ldh [20]					# 加载packet偏移地址为20的半字到寄存器A
	jset #0x1fff, L5, L3		# A&0x1fff是否为真，真去L5，假去L3
L3: ldx 4*([14]&0xf)			# (packet[14]&0xf)*4加载到寄存器X
	ldh [x+16]					# 加载packet偏移地址为x+16的半字到寄存器A
	jeq #N, L4, L5				# 比较port和N是否相等，相等去L4，不等去L5
L4: ret #TRUE					# 返回长度k
L5: ret #0						# 返回长度0
```

下面是动态翻译的源码，只给了几个示例注释：

```c
struct bpf_insn {
	u_short	code;	// 操作码
	u_char 	jt;		// jump ture 时对应的offset
	u_char 	jf;		// jump false时对应的offset
	bpf_int32 k;	// 立即数
};

u_int
bpf_filter(pc, p, wirelen, buflen)
	register struct bpf_insn *pc;
	register u_char *p;
	u_int wirelen;
	register u_int buflen;
{
    // 寄存器A,X
	register u_int32 A, X;
	register int k;
    // scratch memory：临时内存空间
	int32 mem[BPF_MEMWORDS];

	if (pc == 0)
		/*
		 * No filter means accept all.
		 */
		return (u_int)-1;
	A = 0;
	X = 0;
	--pc;
    // 遍历所有的伪指令
	while (1) {
		++pc;
        // 判断操作码类别
		switch (pc->code) {

		default:
#ifdef KERNEL
			return 0;
#else
			abort();
#endif
        // ret指令且指令k字段有效
		case BPF_RET|BPF_K:
			return (u_int)pc->k;
		// ret指令且寄存器A有效
		case BPF_RET|BPF_A:
			return (u_int)A;

		case BPF_LD|BPF_W|BPF_ABS:
			k = pc->k;
			if (k + sizeof(int32) > buflen) {
#ifdef KERNEL
				int merr;

				if (buflen != 0)
					return 0;
				A = m_xword((struct mbuf *)p, k, &merr);
				if (merr != 0)
					return 0;
				continue;
#else
				return 0;
#endif
			}
			A = EXTRACT_LONG(&p[k]);
			continue;

		case BPF_LD|BPF_H|BPF_ABS:
			k = pc->k;
			if (k + sizeof(short) > buflen) {
#ifdef KERNEL
				int merr;

				if (buflen != 0)
					return 0;
				A = m_xhalf((struct mbuf *)p, k, &merr);
				continue;
#else
				return 0;
#endif
			}
			A = EXTRACT_SHORT(&p[k]);
			continue;

		case BPF_LD|BPF_B|BPF_ABS:
			k = pc->k;
			if (k >= buflen) {
#ifdef KERNEL
				register struct mbuf *m;
				register int len;

				if (buflen != 0)
					return 0;
				m = (struct mbuf *)p;
				MINDEX(len, m, k);
				A = mtod(m, u_char *)[k];
				continue;
#else
				return 0;
#endif
			}
			A = p[k];
			continue;

		case BPF_LD|BPF_W|BPF_LEN:
			A = wirelen;
			continue;

		case BPF_LDX|BPF_W|BPF_LEN:
			X = wirelen;
			continue;

		case BPF_LD|BPF_W|BPF_IND:
			k = X + pc->k;
			if (k + sizeof(int32) > buflen) {
#ifdef KERNEL
				int merr;

				if (buflen != 0)
					return 0;
				A = m_xword((struct mbuf *)p, k, &merr);
				if (merr != 0)
					return 0;
				continue;
#else
				return 0;
#endif
			}
			A = EXTRACT_LONG(&p[k]);
			continue;

		case BPF_LD|BPF_H|BPF_IND:
			k = X + pc->k;
			if (k + sizeof(short) > buflen) {
#ifdef KERNEL
				int merr;

				if (buflen != 0)
					return 0;
				A = m_xhalf((struct mbuf *)p, k, &merr);
				if (merr != 0)
					return 0;
				continue;
#else
				return 0;
#endif
			}
			A = EXTRACT_SHORT(&p[k]);
			continue;

		case BPF_LD|BPF_B|BPF_IND:
			k = X + pc->k;
			if (k >= buflen) {
#ifdef KERNEL
				register struct mbuf *m;
				register int len;

				if (buflen != 0)
					return 0;
				m = (struct mbuf *)p;
				MINDEX(len, m, k);
				A = mtod(m, u_char *)[k];
				continue;
#else
				return 0;
#endif
			}
			A = p[k];
			continue;

		case BPF_LDX|BPF_MSH|BPF_B:
			k = pc->k;
			if (k >= buflen) {
#ifdef KERNEL
				register struct mbuf *m;
				register int len;

				if (buflen != 0)
					return 0;
				m = (struct mbuf *)p;
				MINDEX(len, m, k);
				X = (mtod(m, char *)[k] & 0xf) << 2;
				continue;
#else
				return 0;
#endif
			}
			X = (p[pc->k] & 0xf) << 2;
			continue;

		case BPF_LD|BPF_IMM:
			A = pc->k;
			continue;

		case BPF_LDX|BPF_IMM:
			X = pc->k;
			continue;

		case BPF_LD|BPF_MEM:
			A = mem[pc->k];
			continue;
			
		case BPF_LDX|BPF_MEM:
			X = mem[pc->k];
			continue;

		case BPF_ST:
			mem[pc->k] = A;
			continue;

		case BPF_STX:
			mem[pc->k] = X;
			continue;

		case BPF_JMP|BPF_JA:
			pc += pc->k;
			continue;

		case BPF_JMP|BPF_JGT|BPF_K:
			pc += (A > pc->k) ? pc->jt : pc->jf;
			continue;

		case BPF_JMP|BPF_JGE|BPF_K:
			pc += (A >= pc->k) ? pc->jt : pc->jf;
			continue;

		case BPF_JMP|BPF_JEQ|BPF_K:
			pc += (A == pc->k) ? pc->jt : pc->jf;
			continue;

		case BPF_JMP|BPF_JSET|BPF_K:
			pc += (A & pc->k) ? pc->jt : pc->jf;
			continue;

		case BPF_JMP|BPF_JGT|BPF_X:
			pc += (A > X) ? pc->jt : pc->jf;
			continue;

		case BPF_JMP|BPF_JGE|BPF_X:
			pc += (A >= X) ? pc->jt : pc->jf;
			continue;

		case BPF_JMP|BPF_JEQ|BPF_X:
			pc += (A == X) ? pc->jt : pc->jf;
			continue;

		case BPF_JMP|BPF_JSET|BPF_X:
			pc += (A & X) ? pc->jt : pc->jf;
			continue;

		case BPF_ALU|BPF_ADD|BPF_X:
			A += X;
			continue;
			
		case BPF_ALU|BPF_SUB|BPF_X:
			A -= X;
			continue;
			
		case BPF_ALU|BPF_MUL|BPF_X:
			A *= X;
			continue;
			
		case BPF_ALU|BPF_DIV|BPF_X:
			if (X == 0)
				return 0;
			A /= X;
			continue;
			
		case BPF_ALU|BPF_AND|BPF_X:
			A &= X;
			continue;
			
		case BPF_ALU|BPF_OR|BPF_X:
			A |= X;
			continue;

		case BPF_ALU|BPF_LSH|BPF_X:
			A <<= X;
			continue;

		case BPF_ALU|BPF_RSH|BPF_X:
			A >>= X;
			continue;

		case BPF_ALU|BPF_ADD|BPF_K:
			A += pc->k;
			continue;
			
		case BPF_ALU|BPF_SUB|BPF_K:
			A -= pc->k;
			continue;
			
		case BPF_ALU|BPF_MUL|BPF_K:
			A *= pc->k;
			continue;
			
		case BPF_ALU|BPF_DIV|BPF_K:
			A /= pc->k;
			continue;
			
		case BPF_ALU|BPF_AND|BPF_K:
			A &= pc->k;
			continue;
			
		case BPF_ALU|BPF_OR|BPF_K:
			A |= pc->k;
			continue;

		case BPF_ALU|BPF_LSH|BPF_K:
			A <<= pc->k;
			continue;

		case BPF_ALU|BPF_RSH|BPF_K:
			A >>= pc->k;
			continue;

		case BPF_ALU|BPF_NEG:
			A = -A;
			continue;

		case BPF_MISC|BPF_TAX:
			X = A;
			continue;

		case BPF_MISC|BPF_TXA:
			A = X;
			continue;
		}
	}
}
```

### 参考阅读

[1]. Mccanne S , Jacobson V . The BSD Packet Filter: A New Architecture for User-level Packet Capture[C]// Proceedings of the USENIX Winter 1993 Conference Proceedings on USENIX Winter 1993 Conference Proceedings. USENIX Association, 1993.