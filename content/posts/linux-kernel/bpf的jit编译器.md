---
title: "bpf的jit编译器"
date: 2020-05-10T16:33:22+08:00
description: ""
draft: false
tags: [linux内核,linux工具]
categories: [linux内核]
---

2001年，Eric Dumazet往bpf添加了jit（just in time）compiler功能，加速了bpf filter程序运行时间（包含编译+运行）。在[上一篇博文](https://chengshuyi.github.io/posts/linux-kernel/bpf-paper/)，我们知道bpf的原始filter程序是通过翻译一条指令也代表着执行一条指令（没有转换成对应机器的机器码的过程），没有编译的过程。jit编译器可以将bpf的filter程序（伪代码）编译成可以直接在主机运行的机器码。该[patch](https://lwn.net/Articles/437884/)实现了bfp程序到x86_64机器码的转换。

本文简单介绍了jit编译器的原理，并有实例分析。

### jit编译器

jit编译器的出现更多的是为了不同指令集之间的转换，对应着也就是跨平台的实现。以java和c为例，java的运行程序（字节码）可以在任意平台上运行，而c语言运行程序不行。这是因为java在不同的平台都配置有相应的jit编译器。在程序运行的时候，jit编译器会将字节码转换成对应平台的指令，然后执行。

1. 为什么不将程序编译好后运行，而是要运行时编译，再执行？

   静态编译能够对程序做以比jit编译器更大的优化，但是静态编译生成的可执行文件并不能跨平台。jit编译器虽然优化能力有限，但是能够对一条指令在不同的平台实现对应的转换，达到跨平台的目的。

### bpf jit编译器

本节介绍了bpf的jit编译器的具体实现，为了更好的了解jit编译器实现的原理。我们先看一个具体示例，即bpf filter程序和编译后的程序对比；最后我们给出jit编译器的源码，分析整个的工作流程。

bpf filter程序使用的寄存器都是伪寄存器，jit首先要将其与物理寄存器对应起来：

* BPF的accumulator寄存器对应着x86_64平台下的eax寄存器；

* BPF的index寄存器对应着x86_64平台下的ebx寄存器；

* 使用rdi寄存器存放skb的首地址；

* 使用rbp存放frame的首地址；

* r9存放头部长度；

* r8存放数据的首地址；

```
# jit filter程序
(000) ldh      [12]
(001) jeq      #0x800           jt 2	jf 6
(002) ld       [26]
(003) jeq      #0xc0a80001      jt 12	jf 4
(004) ld       [30]
(005) jeq      #0xc0a80001      jt 12	jf 13
(006) jeq      #0x806           jt 8	jf 7
(007) jeq      #0x8035          jt 8	jf 13
(008) ld       [28]
(009) jeq      #0xc0a80001      jt 12	jf 10
(010) ld       [38]
(011) jeq      #0xc0a80001      jt 12	jf 13
(012) ret      #65535
(013) ret      #0
# jit编译后的机器码
JIT code: ffffffffa00d5000: 55 48 89 e5 48 83 ec 60 48 89 5d f8 44 8b 4f 60
JIT code: ffffffffa00d5010: 44 2b 4f 64 4c 8b 87 b8 00 00 00 be 0c 00 00 00
JIT code: ffffffffa00d5020: e8 24 1b 2f e1 3d 00 08 00 00 75 24 be 1a 00 00
JIT code: ffffffffa00d5030: 00 e8 fe 1a 2f e1 3d 01 00 a8 c0 74 43 be 1e 00
JIT code: ffffffffa00d5040: 00 00 e8 ed 1a 2f e1 3d 01 00 a8 c0 74 32 eb 37
JIT code: ffffffffa00d5050: 3d 06 08 00 00 74 07 3d 35 80 00 00 75 29 be 1c
JIT code: ffffffffa00d5060: 00 00 00 e8 cc 1a 2f e1 3d 01 00 a8 c0 74 11 be
JIT code: ffffffffa00d5070: 26 00 00 00 e8 bb 1a 2f e1 3d 01 00 a8 c0 75 07
JIT code: ffffffffa00d5080: b8 ff ff 00 00 eb 02 31 c0 c9 c3
# 机器码转换成便于分析的汇编代码
0000: push %rbp
	  mov %rsp,%rbp
0004: subq $96,%rsp
0008: mov %rbx, -8(%rbp)
000c: mov off8(%rdi),%r9d
0010: sub off8(%rdi),%r9d
0014: mov off32(%rdi),%r8   b8 00 00 00
001c: mov 0x0c,%esi
0020: call function(addr is 0xe12f1b24)
0025: cmp #0x800,%eax
002a: jne 0x24
002c: mov $26,%esi
0031: call function(addr is 0xe12f1afe)
0036: cmp $0xc0a80001,%eax
003b: je 0x43
003d: mov $30,%esi
0042: call function(addr is 0xe12f1aed)
0047: cmp $0xc0a80001,%eax
004c: je 0x32
004e: jmp 0x37
0050: cmp $0x806,%eax
0055: je 0x07
0057: cmp $0x8035,%eax
005d: jne 0x29
005e: jmp 0x1c
0063: call function(addr is 0xe12f1acc)
0068: cmp $0xc0a80001,%eax
006d: je 0x11
006f: mov 0x26,%esi 
0074: call function(addr is 0xe12f1abb)
0079: cmp $0xc0a80001,%eax
007e: jne 0x07
mov $65535,%eax 
jmp 02
xor %eax,%eax
leaveq
ret
```

下面是jit编译的源码，有删除一部分。同时，给了一部分的注释：

```c
#include <linux/moduleloader.h>
#include <asm/cacheflush.h>
#include <linux/netdevice.h>
#include <linux/filter.h>
/*
 * Conventions :
 *  EAX : BPF A accumulator
 *  EBX : BPF X accumulator
 *  RDI : pointer to skb   (first argument given to JIT function)
 *  RBP : frame pointer (even if CONFIG_FRAME_POINTER=n)
 *  r9d : skb->len - skb->data_len (headlen)
 *  r8  : skb->data
 */
int bpf_jit_enable __read_mostly;

/*
 * assembly code in net/core/bpf_jit.S
 */
extern u8 sk_load_word[], sk_load_half[], sk_load_byte[], sk_load_byte_msh[];
extern u8 sk_load_word_ind[], sk_load_half_ind[], sk_load_byte_ind[];

static inline u8 *emit_code(u8 *ptr, u32 bytes, unsigned int len)
{
	if (len == 1)
		*ptr = bytes;
	else if (len == 2)
		*(u16 *)ptr = bytes;
	else {
		*(u32 *)ptr = bytes;
		barrier();
	}
	return ptr + len;
}

#define EMIT(bytes, len)	do { prog = emit_code(prog, bytes, len); } while (0)

#define EMIT1(b1)		EMIT(b1, 1)
#define EMIT2(b1, b2)		EMIT((b1) + ((b2) << 8), 2)
#define EMIT3(b1, b2, b3)	EMIT((b1) + ((b2) << 8) + ((b3) << 16), 3)
#define EMIT4(b1, b2, b3, b4)   EMIT((b1) + ((b2) << 8) + ((b3) << 16) + ((b4) << 24), 4)
#define EMIT1_off32(b1, off)	do { EMIT1(b1); EMIT(off, 4);} while (0)

#define CLEAR_A() EMIT2(0x31, 0xc0) /* xor %eax,%eax */
#define CLEAR_X() EMIT2(0x31, 0xdb) /* xor %ebx,%ebx */

static inline bool is_imm8(int value)
{
	return value <= 127 && value >= -128;
}

static inline bool is_near(int offset)
{
	return offset <= 127 && offset >= -128;
}

#define EMIT_JMP(offset)						\
do {									\
	if (offset) {							\
		if (is_near(offset))					\
			EMIT2(0xeb, offset); /* jmp .+off8 */		\
		else							\
			EMIT1_off32(0xe9, offset); /* jmp .+off32 */	\
	}								\
} while (0)

/* list of x86 cond jumps (. + s8)
 * Add 0x10 (and an extra 0x0f) to generate far jumps (. + s32)
 */
#define X86_JB  0x72
#define X86_JAE 0x73
#define X86_JE  0x74
#define X86_JNE 0x75
#define X86_JBE 0x76
#define X86_JA  0x77

#define EMIT_COND_JMP(op, offset)				\
do {								\
	if (is_near(offset))					\
		EMIT2(op, offset); /* jxx .+off8 */		\
	else {							\
		EMIT2(0x0f, op + 0x10);				\
		EMIT(offset, 4); /* jxx .+off32 */		\
	}							\
} while (0)

#define COND_SEL(CODE, TOP, FOP)	\
	case CODE:			\
		t_op = TOP;		\
		f_op = FOP;		\
		goto cond_branch


void bpf_jit_compile(struct sk_filter *fp)
{
	u8 temp[64];
	// 编译后生成的机器码
	u8 *prog;
	unsigned int proglen, oldproglen = 0;
	int ilen, i;
	int t_offset, f_offset;
	u8 t_op, f_op, seen = 0, pass;
	u8 *image = NULL;
	u8 *func;
	int pc_ret0 = -1; /* bpf index of first RET #0 instruction (if any) */
	unsigned int cleanup_addr;
	unsigned int *addrs;
	const struct sock_filter *filter = fp->insns;
	// filter程序的指令个数
	int flen = fp->len;

	if (!bpf_jit_enable)
		return;
	addrs = kmalloc(flen * sizeof(*addrs), GFP_KERNEL);
	if (addrs == NULL)
		return;

	/* Before first pass, make a rough estimation of addrs[]
	 * each bpf instruction is translated to less than 64 bytes
	 */
	for (proglen = 0, i = 0; i < flen; i++) {
		proglen += 64;
		addrs[i] = proglen;
	}
	cleanup_addr = proglen; /* epilogue address */

	for (pass = 0; pass < 10; pass++) {
		/* no prologue/epilogue for trivial filters (RET something) */
		proglen = 0;
		prog = temp;

		// 第一遍不做翻译，只是浏览整个程序是否包含某些指令，主要是为了处理prologue和epilogue
		if (seen) {
		    // prologue
			EMIT4(0x55, 0x48, 0x89, 0xe5); /* push %rbp; mov %rsp,%rbp */
			EMIT4(0x48, 0x83, 0xec, 96);	/* subq  $96,%rsp	*/
			/* note : must save %rbx in case bpf_error is hit */
			// 保存callee register
			if (seen & (SEEN_XREG | SEEN_DATAREF))
				EMIT4(0x48, 0x89, 0x5d, 0xf8); /* mov %rbx, -8(%rbp) */
			if (seen & SEEN_XREG)
				CLEAR_X(); /* make sure we dont leek kernel memory */
			// 传递参数
			if (seen & SEEN_DATAREF) {
				if (offsetof(struct sk_buff, len) <= 127)
					/* mov    off8(%rdi),%r9d */
					EMIT4(0x44, 0x8b, 0x4f, offsetof(struct sk_buff, len));
				else {
					/* mov    off32(%rdi),%r9d */
					EMIT3(0x44, 0x8b, 0x8f);
					EMIT(offsetof(struct sk_buff, len), 4);
				}
				if (is_imm8(offsetof(struct sk_buff, data_len)))
					/* sub    off8(%rdi),%r9d */
					EMIT4(0x44, 0x2b, 0x4f, offsetof(struct sk_buff, data_len));
				else {
					EMIT3(0x44, 0x2b, 0x8f);
					EMIT(offsetof(struct sk_buff, data_len), 4);
				}

				if (is_imm8(offsetof(struct sk_buff, data)))
					/* mov off8(%rdi),%r8 */
					EMIT4(0x4c, 0x8b, 0x47, offsetof(struct sk_buff, data));
				else {
					/* mov off32(%rdi),%r8 */
					EMIT3(0x4c, 0x8b, 0x87);
					EMIT(offsetof(struct sk_buff, data), 4);
				}
			}
		}
        // 解析bpf的第一条指令
		switch (filter[0].code) {
		case BPF_S_RET_K:
		case BPF_S_LD_W_LEN:
		case BPF_S_ANC_PROTOCOL:
		case BPF_S_ANC_IFINDEX:
		case BPF_S_ANC_MARK:
		case BPF_S_ANC_RXHASH:
		case BPF_S_ANC_CPU:
		case BPF_S_LD_W_ABS:
		case BPF_S_LD_H_ABS:
		case BPF_S_LD_B_ABS:
			/* first instruction sets A register (or is RET 'constant') */
			break;
		default:
			CLEAR_A(); /* A = 0 */
		}
        // 编译每一条bpf filter程序指令
		for (i = 0; i < flen; i++) {
			unsigned int K = filter[i].k;
            // 解析bpf指令的操作码
			switch (filter[i].code) {
			case BPF_S_ALU_ADD_X: /* A += X; */
				seen |= SEEN_XREG;
				EMIT2(0x01, 0xd8);		/* add %ebx,%eax */
				break;
			case BPF_S_ALU_ADD_K: /* A += K; */
				if (!K)
					break;
				if (is_imm8(K))
					EMIT3(0x83, 0xc0, K);	/* add imm8,%eax */
				else
					EMIT1_off32(0x05, K);	/* add imm32,%eax */
				break;
			case BPF_S_ALU_MUL_X: /* A *= X; */
				seen |= SEEN_XREG;
				EMIT3(0x0f, 0xaf, 0xc3);	/* imul %ebx,%eax */
				break;

			case BPF_S_RET_K:
				if (!K) {
					if (pc_ret0 == -1)
						pc_ret0 = i;
					CLEAR_A();
				} else {
					EMIT1_off32(0xb8, K);	/* mov $imm32,%eax */
				}
				/* fallinto */
			case BPF_S_RET_A:
				if (seen) {
					if (i != flen - 1) {
						EMIT_JMP(cleanup_addr - addrs[i]);
						break;
					}
					if (seen & SEEN_XREG)
						EMIT4(0x48, 0x8b, 0x5d, 0xf8);  /* mov  -8(%rbp),%rbx */
					EMIT1(0xc9);		/* leaveq */
				}
				EMIT1(0xc3);		/* ret */
				break;
			case BPF_S_LD_MEM: /* A = mem[K] : mov off8(%rbp),%eax */
				seen |= SEEN_MEM;
				EMIT3(0x8b, 0x45, 0xf0 - K*4);
				break;
			case BPF_S_LDX_MEM: /* X = mem[K] : mov off8(%rbp),%ebx */
				seen |= SEEN_XREG | SEEN_MEM;
				EMIT3(0x8b, 0x5d, 0xf0 - K*4);
				break;
			case BPF_S_LD_W_LEN: /*	A = skb->len; */
				BUILD_BUG_ON(FIELD_SIZEOF(struct sk_buff, len) != 4);
				if (is_imm8(offsetof(struct sk_buff, len)))
					/* mov    off8(%rdi),%eax */
					EMIT3(0x8b, 0x47, offsetof(struct sk_buff, len));
				else {
					EMIT2(0x8b, 0x87);
					EMIT(offsetof(struct sk_buff, len), 4);
				}
				break;
			case BPF_S_LDX_W_LEN: /* X = skb->len; */
				seen |= SEEN_XREG;
				if (is_imm8(offsetof(struct sk_buff, len)))
					/* mov off8(%rdi),%ebx */
					EMIT3(0x8b, 0x5f, offsetof(struct sk_buff, len));
				else {
					EMIT2(0x8b, 0x9f);
					EMIT(offsetof(struct sk_buff, len), 4);
				}
				break;
			case BPF_S_ANC_CPU:
#ifdef CONFIG_SMP
				EMIT4(0x65, 0x8b, 0x04, 0x25); /* mov %gs:off32,%eax */
				EMIT((u32)&cpu_number, 4); /* A = smp_processor_id(); */
#else
				CLEAR_A();
#endif
				break;
			case BPF_S_LD_W_ABS:
				func = sk_load_word;
common_load:			seen |= SEEN_DATAREF;
				if ((int)K < 0)
					goto out;
				t_offset = func - (image + addrs[i]);
				EMIT1_off32(0xbe, K); /* mov imm32,%esi */
				EMIT1_off32(0xe8, t_offset); /* call */
				break;
			case BPF_S_LD_H_ABS:
				func = sk_load_half;
				goto common_load;
			case BPF_S_LD_B_ABS:
				func = sk_load_byte;
				goto common_load;
			case BPF_S_LDX_B_MSH:
				if ((int)K < 0) {
					if (pc_ret0 != -1) {
						EMIT_JMP(addrs[pc_ret0] - addrs[i]);
						break;
					}
					CLEAR_A();
					EMIT_JMP(cleanup_addr - addrs[i]);
					break;
				}
				seen |= SEEN_DATAREF | SEEN_XREG;
				t_offset = sk_load_byte_msh - (image + addrs[i]);
				EMIT1_off32(0xbe, K);	/* mov imm32,%esi */
				EMIT1_off32(0xe8, t_offset); /* call sk_load_byte_msh */
				break;
			case BPF_S_LD_W_IND:
				func = sk_load_word_ind;
common_load_ind:		seen |= SEEN_DATAREF | SEEN_XREG;
				t_offset = func - (image + addrs[i]);
				EMIT1_off32(0xbe, K);	/* mov imm32,%esi   */
				EMIT1_off32(0xe8, t_offset);	/* call sk_load_xxx_ind */
				break;
			case BPF_S_LD_H_IND:
				func = sk_load_half_ind;
				goto common_load_ind;
			case BPF_S_LD_B_IND:
				func = sk_load_byte_ind;
				goto common_load_ind;
			case BPF_S_JMP_JA:
				t_offset = addrs[i + K] - addrs[i];
				EMIT_JMP(t_offset);
				break;
			COND_SEL(BPF_S_JMP_JGT_K, X86_JA, X86_JBE);
			COND_SEL(BPF_S_JMP_JGE_K, X86_JAE, X86_JB);
			COND_SEL(BPF_S_JMP_JEQ_K, X86_JE, X86_JNE);
			COND_SEL(BPF_S_JMP_JSET_K,X86_JNE, X86_JE);
			COND_SEL(BPF_S_JMP_JGT_X, X86_JA, X86_JBE);
			COND_SEL(BPF_S_JMP_JGE_X, X86_JAE, X86_JB);
			COND_SEL(BPF_S_JMP_JEQ_X, X86_JE, X86_JNE);
			COND_SEL(BPF_S_JMP_JSET_X,X86_JNE, X86_JE);

cond_branch:
			    f_offset = addrs[i + filter[i].jf] - addrs[i];
				t_offset = addrs[i + filter[i].jt] - addrs[i];

				/* same targets, can avoid doing the test :) */
				if (filter[i].jt == filter[i].jf) {
					EMIT_JMP(t_offset);
					break;
				}

				switch (filter[i].code) {
				case BPF_S_JMP_JGT_X:
				case BPF_S_JMP_JGE_X:
				case BPF_S_JMP_JEQ_X:
					seen |= SEEN_XREG;
					EMIT2(0x39, 0xd8); /* cmp %ebx,%eax */
					break;
				case BPF_S_JMP_JSET_X:
					seen |= SEEN_XREG;
					EMIT2(0x85, 0xd8); /* test %ebx,%eax */
					break;
				case BPF_S_JMP_JEQ_K:
					if (K == 0) {
						EMIT2(0x85, 0xc0); /* test   %eax,%eax */
						break;
					}
				case BPF_S_JMP_JGT_K:
				case BPF_S_JMP_JGE_K:
					if (K <= 127)
						EMIT3(0x83, 0xf8, K); /* cmp imm8,%eax */
					else
						EMIT1_off32(0x3d, K); /* cmp imm32,%eax */
					break;
				case BPF_S_JMP_JSET_K:
					if (K <= 0xFF)
						EMIT2(0xa8, K); /* test imm8,%al */
					else if (!(K & 0xFFFF00FF))
						EMIT3(0xf6, 0xc4, K >> 8); /* test imm8,%ah */
					else if (K <= 0xFFFF) {
						EMIT2(0x66, 0xa9); /* test imm16,%ax */
						EMIT(K, 2);
					} else {
						EMIT1_off32(0xa9, K); /* test imm32,%eax */
					}
					break;
				}
				if (filter[i].jt != 0) {
					if (filter[i].jf)
						t_offset += is_near(f_offset) ? 2 : 6;
					EMIT_COND_JMP(t_op, t_offset);
					if (filter[i].jf)
						EMIT_JMP(f_offset);
					break;
				}
				EMIT_COND_JMP(f_op, f_offset);
				break;
			default:
				/* hmm, too complex filter, give up with jit compiler */
				goto out;
			}
			// bpf一条指令编译后的指令长度
			ilen = prog - temp;
			if (image) {
				if (unlikely(proglen + ilen > oldproglen)) {
					pr_err("bpb_jit_compile fatal error\n");
					kfree(addrs);
					module_free(NULL, image);
					return;
				}
				memcpy(image + proglen, temp, ilen);
			}
			proglen += ilen;
			addrs[i] = proglen;
			prog = temp;
		}
		/* last bpf instruction is always a RET :
		 * use it to give the cleanup instruction(s) addr
		 */
		// 处理epilogue
		cleanup_addr = proglen - 1; /* ret */
		// 恢复上一个函数的rbp和rsp
		// movq %rbp, %rsp
        // popq %rbp
		if (seen)
			cleanup_addr -= 1; /* leaveq */
        // 恢复rbx寄存器的值
		if (seen & SEEN_XREG)
			cleanup_addr -= 4; /* mov  -8(%rbp),%rbx */

		if (image) {
		    // 编译完成，退出
		    // pass = 2
			WARN_ON(proglen != oldproglen);
			break;
		}
		// pass = 1
		if (proglen == oldproglen) {
		    // 为代码分配内存（主要是设置page的属性为可执行，也就是指令而非数据）
		    // 这也是动态模块加载可以运行的基本条件
			image = module_alloc(max_t(unsigned int,proglen,sizeof(struct work_struct)));
			if (!image)
				goto out;
		}
		// pass = 0
		oldproglen = proglen;
	}
	if (bpf_jit_enable > 1)
		pr_err("flen=%d proglen=%u pass=%d image=%p\n",
		       flen, proglen, pass, image);

	if (image) {
		if (bpf_jit_enable > 1)
			print_hex_dump(KERN_ERR, "JIT code: ", DUMP_PREFIX_ADDRESS,
				       16, 1, image, proglen, false);
        // 刷新代码到内存
		bpf_flush_icache(image, image + proglen);

		fp->bpf_func = (void *)image;
	}
out:
	kfree(addrs);
	return;
}

```

### 参考文章

[1]. Mccanne S , Jacobson V . The BSD Packet Filter: A New Architecture for User-level Packet Capture[C]// Proceedings of the USENIX Winter 1993 Conference Proceedings on USENIX Winter 1993 Conference Proceedings. USENIX Association, 1993.

[2]. A JIT for packet filters. https://lwn.net/Articles/437981/.

[3]. net: filter: Just In Time compiler. https://lwn.net/Articles/437884/.

[4]. Understanding JIT compiler (just-in-time compiler) for java. https://aboullaite.me/understanding-jit-compiler-just-in-time-compiler/.

