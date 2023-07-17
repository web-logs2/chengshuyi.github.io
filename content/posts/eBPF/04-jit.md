---
title: "eBPF 解释器和JIT"
date: 2023-05-01T14:08:58+08:00 
description: ""
draft: false
tags: [eBPF,eBPF JIT]
categories: [eBPF]
---

在eBPF中，有两种执行eBPF程序的方式：解释器（interpreter）和JIT（Just-In-Time）编译器。

当eBPF程序需要执行时，内核会调用解释器来执行程序。解释器将逐条解释和执行eBPF指令，最终输出执行结果。解释器的优点是简单易用，能够实现快速开发和调试，同时也具有较好的可移植性。但是，解释器的执行速度相对较慢，特别是对于复杂的eBPF程序，执行效率会更低。相比之下，jit编译器可以将eBPF程序编译成本地机器码，从而实现快速执行。jit编译器在eBPF程序第一次加载时会对程序进行编译，并将编译结果缓存起来，下次执行时可以直接使用编译结果，从而大大提高了程序的执行效率。jit编译器的缺点是需要消耗一定的时间和资源来进行编译，同时也可能会出现一些性能问题。
<!-- 
总的来说，eBPF解释器适合开发和调试复杂的eBPF程序，而jit编译器适合在生产环境中执行eBPF程序，以获得更好的执行性能。

eBPF JIT（Just-In-Time）是一种即时编译技术，它能够将eBPF程序字节码转换为机器码。在运行时，JIT编译器会分析eBPF程序并将其转换为机器指令，从而加速eBPF程序的执行速度。JIT编译器可以将eBPF程序优化为特定的CPU架构和操作系统，从而提高程序的性能和效率。此外，JIT编译器还可以对eBPF程序进行安全检查，确保它们不会对系统造成任何损害。 -->



截止目前，eBPF的JIT已经支持了多个架构：
arm
arm64
loongarch
mips
powerpc
riscv
s390
sparc
x86
x86-64


### 工作流程

bpf_prog_load -》 bpf_prog_select_runtime -》bpf_int_jit_compile
											-》 bpf_prog_select_func
虽然eBPF解释器和JIT编译器都可以在程序运行时动态执行代码，但它们的机制和应用场景不同。eBPF解释器主要用于在内核中运行自定义程序，而JIT编译器主要用于优化动态语言和虚拟机中的代码执行速度。
	当内核开启时CONFIG_BPF_JIT_ALWAYS_ON，会进行JIT，反之则会使用eBPF解释器运行eBPF程序。

### eBPF 解释器

eBPF解释器是一种虚拟机，在Linux内核中被用于运行自定义的、安全的程序。eBPF程序可以编写为类C语言代码，然后使用编译器将其编译为eBPF指令集。eBPF解释器使用这些指令来执行程序，并与内核中的网络、存储和安全子系统进行交互。eBPF解释器的执行速度较快，安全性高，并且提供了丰富的API和工具，方便用户开发和调试自己的eBPF程序。


```c

static void bpf_prog_select_func(struct bpf_prog *fp)
{
	u32 stack_depth = max_t(u32, fp->aux->stack_depth, 1);
	fp->bpf_func = interpreters[(round_up(stack_depth, 32) / 32) - 1];
}

static unsigned int (*interpreters[])(const void *ctx,
				      const struct bpf_insn *insn) = {
EVAL6(PROG_NAME_LIST, 32, 64, 96, 128, 160, 192)
EVAL6(PROG_NAME_LIST, 224, 256, 288, 320, 352, 384)
EVAL4(PROG_NAME_LIST, 416, 448, 480, 512)
};

#define PROG_NAME_LIST(stack_size) PROG_NAME(stack_size),

#define PROG_NAME(stack_size) __bpf_prog_run##stack_size

#define DEFINE_BPF_PROG_RUN(stack_size) \
static unsigned int PROG_NAME(stack_size)(const void *ctx, const struct bpf_insn *insn) \
{ \
	u64 stack[stack_size / sizeof(u64)]; \
	u64 regs[MAX_BPF_EXT_REG] = {}; \
\
	FP = (u64) (unsigned long) &stack[ARRAY_SIZE(stack)]; \
	ARG1 = (u64) (unsigned long) ctx; \
	return ___bpf_prog_run(regs, insn); \
}

static u64 ___bpf_prog_run(u64 *regs, const struct bpf_insn *insn)
{
#define BPF_INSN_2_LBL(x, y)    [BPF_##x | BPF_##y] = &&x##_##y
#define BPF_INSN_3_LBL(x, y, z) [BPF_##x | BPF_##y | BPF_##z] = &&x##_##y##_##z
	static const void * const jumptable[256] __annotate_jump_table = {
		[0 ... 255] = &&default_label,
		/* Now overwrite non-defaults ... */
		BPF_INSN_MAP(BPF_INSN_2_LBL, BPF_INSN_3_LBL),
		/* Non-UAPI available opcodes. */
		[BPF_JMP | BPF_CALL_ARGS] = &&JMP_CALL_ARGS,
		[BPF_JMP | BPF_TAIL_CALL] = &&JMP_TAIL_CALL,
		[BPF_ST  | BPF_NOSPEC] = &&ST_NOSPEC,
		[BPF_LDX | BPF_PROBE_MEM | BPF_B] = &&LDX_PROBE_MEM_B,
		[BPF_LDX | BPF_PROBE_MEM | BPF_H] = &&LDX_PROBE_MEM_H,
		[BPF_LDX | BPF_PROBE_MEM | BPF_W] = &&LDX_PROBE_MEM_W,
		[BPF_LDX | BPF_PROBE_MEM | BPF_DW] = &&LDX_PROBE_MEM_DW,
	};
#undef BPF_INSN_3_LBL
#undef BPF_INSN_2_LBL
	u32 tail_call_cnt = 0;

#define CONT	 ({ insn++; goto select_insn; })
#define CONT_JMP ({ insn++; goto select_insn; })

select_insn:
	goto *jumptable[insn->code];

	/* Explicitly mask the register-based shift amounts with 63 or 31
	 * to avoid undefined behavior. Normally this won't affect the
	 * generated code, for example, in case of native 64 bit archs such
	 * as x86-64 or arm64, the compiler is optimizing the AND away for
	 * the interpreter. In case of JITs, each of the JIT backends compiles
	 * the BPF shift operations to machine instructions which produce
	 * implementation-defined results in such a case; the resulting
	 * contents of the register may be arbitrary, but program behaviour
	 * as a whole remains defined. In other words, in case of JIT backends,
	 * the AND must /not/ be added to the emitted LSH/RSH/ARSH translation.
	 */
	/* ALU (shifts) */
#define SHT(OPCODE, OP)					\
	ALU64_##OPCODE##_X:				\
		DST = DST OP (SRC & 63);		\
		CONT;					\
	ALU_##OPCODE##_X:				\
		DST = (u32) DST OP ((u32) SRC & 31);	\
		CONT;					\
	ALU64_##OPCODE##_K:				\
		DST = DST OP IMM;			\
		CONT;					\
	ALU_##OPCODE##_K:				\
		DST = (u32) DST OP (u32) IMM;		\
		CONT;
	/* ALU (rest) */
#define ALU(OPCODE, OP)					\
	ALU64_##OPCODE##_X:				\
		DST = DST OP SRC;			\
		CONT;					\
	ALU_##OPCODE##_X:				\
		DST = (u32) DST OP (u32) SRC;		\
		CONT;					\
	ALU64_##OPCODE##_K:				\
		DST = DST OP IMM;			\
		CONT;					\
	ALU_##OPCODE##_K:				\
		DST = (u32) DST OP (u32) IMM;		\
		CONT;
	ALU(ADD,  +)
	ALU(SUB,  -)
	ALU(AND,  &)
	ALU(OR,   |)
	ALU(XOR,  ^)
	ALU(MUL,  *)
	SHT(LSH, <<)
	SHT(RSH, >>)
#undef SHT
#undef ALU
	ALU_NEG:
		DST = (u32) -DST;
		CONT;
	ALU64_NEG:
		DST = -DST;
		CONT;
	ALU_MOV_X:
		DST = (u32) SRC;
		CONT;
	ALU_MOV_K:
		DST = (u32) IMM;
		CONT;
	ALU64_MOV_X:
		DST = SRC;
		CONT;
	ALU64_MOV_K:
		DST = IMM;
		CONT;
	LD_IMM_DW:
		DST = (u64) (u32) insn[0].imm | ((u64) (u32) insn[1].imm) << 32;
		insn++;
		CONT;
	ALU_ARSH_X:
		DST = (u64) (u32) (((s32) DST) >> (SRC & 31));
		CONT;
	ALU_ARSH_K:
		DST = (u64) (u32) (((s32) DST) >> IMM);
		CONT;
	ALU64_ARSH_X:
		(*(s64 *) &DST) >>= (SRC & 63);
		CONT;
	ALU64_ARSH_K:
		(*(s64 *) &DST) >>= IMM;
		CONT;
	ALU64_MOD_X:
		div64_u64_rem(DST, SRC, &AX);
		DST = AX;
		CONT;
	ALU_MOD_X:
		AX = (u32) DST;
		DST = do_div(AX, (u32) SRC);
		CONT;
	ALU64_MOD_K:
		div64_u64_rem(DST, IMM, &AX);
		DST = AX;
		CONT;
	ALU_MOD_K:
		AX = (u32) DST;
		DST = do_div(AX, (u32) IMM);
		CONT;
	ALU64_DIV_X:
		DST = div64_u64(DST, SRC);
		CONT;
	ALU_DIV_X:
		AX = (u32) DST;
		do_div(AX, (u32) SRC);
		DST = (u32) AX;
		CONT;
	ALU64_DIV_K:
		DST = div64_u64(DST, IMM);
		CONT;
	ALU_DIV_K:
		AX = (u32) DST;
		do_div(AX, (u32) IMM);
		DST = (u32) AX;
		CONT;
	ALU_END_TO_BE:
		switch (IMM) {
		case 16:
			DST = (__force u16) cpu_to_be16(DST);
			break;
		case 32:
			DST = (__force u32) cpu_to_be32(DST);
			break;
		case 64:
			DST = (__force u64) cpu_to_be64(DST);
			break;
		}
		CONT;
	ALU_END_TO_LE:
		switch (IMM) {
		case 16:
			DST = (__force u16) cpu_to_le16(DST);
			break;
		case 32:
			DST = (__force u32) cpu_to_le32(DST);
			break;
		case 64:
			DST = (__force u64) cpu_to_le64(DST);
			break;
		}
		CONT;

	/* CALL */
	JMP_CALL:
		/* Function call scratches BPF_R1-BPF_R5 registers,
		 * preserves BPF_R6-BPF_R9, and stores return value
		 * into BPF_R0.
		 */
		BPF_R0 = (__bpf_call_base + insn->imm)(BPF_R1, BPF_R2, BPF_R3,
						       BPF_R4, BPF_R5);
		CONT;

	JMP_CALL_ARGS:
		BPF_R0 = (__bpf_call_base_args + insn->imm)(BPF_R1, BPF_R2,
							    BPF_R3, BPF_R4,
							    BPF_R5,
							    insn + insn->off + 1);
		CONT;

	JMP_TAIL_CALL: {
		struct bpf_map *map = (struct bpf_map *) (unsigned long) BPF_R2;
		struct bpf_array *array = container_of(map, struct bpf_array, map);
		struct bpf_prog *prog;
		u32 index = BPF_R3;

		if (unlikely(index >= array->map.max_entries))
			goto out;

		if (unlikely(tail_call_cnt >= MAX_TAIL_CALL_CNT))
			goto out;

		tail_call_cnt++;

		prog = READ_ONCE(array->ptrs[index]);
		if (!prog)
			goto out;

		/* ARG1 at this point is guaranteed to point to CTX from
		 * the verifier side due to the fact that the tail call is
		 * handled like a helper, that is, bpf_tail_call_proto,
		 * where arg1_type is ARG_PTR_TO_CTX.
		 */
		insn = prog->insnsi;
		goto select_insn;
out:
		CONT;
	}
	JMP_JA:
		insn += insn->off;
		CONT;
	JMP_EXIT:
		return BPF_R0;
	/* JMP */
#define COND_JMP(SIGN, OPCODE, CMP_OP)				\
	JMP_##OPCODE##_X:					\
		if ((SIGN##64) DST CMP_OP (SIGN##64) SRC) {	\
			insn += insn->off;			\
			CONT_JMP;				\
		}						\
		CONT;						\
	JMP32_##OPCODE##_X:					\
		if ((SIGN##32) DST CMP_OP (SIGN##32) SRC) {	\
			insn += insn->off;			\
			CONT_JMP;				\
		}						\
		CONT;						\
	JMP_##OPCODE##_K:					\
		if ((SIGN##64) DST CMP_OP (SIGN##64) IMM) {	\
			insn += insn->off;			\
			CONT_JMP;				\
		}						\
		CONT;						\
	JMP32_##OPCODE##_K:					\
		if ((SIGN##32) DST CMP_OP (SIGN##32) IMM) {	\
			insn += insn->off;			\
			CONT_JMP;				\
		}						\
		CONT;
	COND_JMP(u, JEQ, ==)
	COND_JMP(u, JNE, !=)
	COND_JMP(u, JGT, >)
	COND_JMP(u, JLT, <)
	COND_JMP(u, JGE, >=)
	COND_JMP(u, JLE, <=)
	COND_JMP(u, JSET, &)
	COND_JMP(s, JSGT, >)
	COND_JMP(s, JSLT, <)
	COND_JMP(s, JSGE, >=)
	COND_JMP(s, JSLE, <=)
#undef COND_JMP
	/* ST, STX and LDX*/
	ST_NOSPEC:
		/* Speculation barrier for mitigating Speculative Store Bypass.
		 * In case of arm64, we rely on the firmware mitigation as
		 * controlled via the ssbd kernel parameter. Whenever the
		 * mitigation is enabled, it works for all of the kernel code
		 * with no need to provide any additional instructions here.
		 * In case of x86, we use 'lfence' insn for mitigation. We
		 * reuse preexisting logic from Spectre v1 mitigation that
		 * happens to produce the required code on x86 for v4 as well.
		 */
		barrier_nospec();
		CONT;
#define LDST(SIZEOP, SIZE)						\
	STX_MEM_##SIZEOP:						\
		*(SIZE *)(unsigned long) (DST + insn->off) = SRC;	\
		CONT;							\
	ST_MEM_##SIZEOP:						\
		*(SIZE *)(unsigned long) (DST + insn->off) = IMM;	\
		CONT;							\
	LDX_MEM_##SIZEOP:						\
		DST = *(SIZE *)(unsigned long) (SRC + insn->off);	\
		CONT;							\
	LDX_PROBE_MEM_##SIZEOP:						\
		bpf_probe_read_kernel(&DST, sizeof(SIZE),		\
				      (const void *)(long) (SRC + insn->off));	\
		DST = *((SIZE *)&DST);					\
		CONT;

	LDST(B,   u8)
	LDST(H,  u16)
	LDST(W,  u32)
	LDST(DW, u64)
#undef LDST

#define ATOMIC_ALU_OP(BOP, KOP)						\
		case BOP:						\
			if (BPF_SIZE(insn->code) == BPF_W)		\
				atomic_##KOP((u32) SRC, (atomic_t *)(unsigned long) \
					     (DST + insn->off));	\
			else						\
				atomic64_##KOP((u64) SRC, (atomic64_t *)(unsigned long) \
					       (DST + insn->off));	\
			break;						\
		case BOP | BPF_FETCH:					\
			if (BPF_SIZE(insn->code) == BPF_W)		\
				SRC = (u32) atomic_fetch_##KOP(		\
					(u32) SRC,			\
					(atomic_t *)(unsigned long) (DST + insn->off)); \
			else						\
				SRC = (u64) atomic64_fetch_##KOP(	\
					(u64) SRC,			\
					(atomic64_t *)(unsigned long) (DST + insn->off)); \
			break;

	STX_ATOMIC_DW:
	STX_ATOMIC_W:
		switch (IMM) {
		ATOMIC_ALU_OP(BPF_ADD, add)
		ATOMIC_ALU_OP(BPF_AND, and)
		ATOMIC_ALU_OP(BPF_OR, or)
		ATOMIC_ALU_OP(BPF_XOR, xor)
#undef ATOMIC_ALU_OP

		case BPF_XCHG:
			if (BPF_SIZE(insn->code) == BPF_W)
				SRC = (u32) atomic_xchg(
					(atomic_t *)(unsigned long) (DST + insn->off),
					(u32) SRC);
			else
				SRC = (u64) atomic64_xchg(
					(atomic64_t *)(unsigned long) (DST + insn->off),
					(u64) SRC);
			break;
		case BPF_CMPXCHG:
			if (BPF_SIZE(insn->code) == BPF_W)
				BPF_R0 = (u32) atomic_cmpxchg(
					(atomic_t *)(unsigned long) (DST + insn->off),
					(u32) BPF_R0, (u32) SRC);
			else
				BPF_R0 = (u64) atomic64_cmpxchg(
					(atomic64_t *)(unsigned long) (DST + insn->off),
					(u64) BPF_R0, (u64) SRC);
			break;

		default:
			goto default_label;
		}
		CONT;

	default_label:
		/* If we ever reach this, we have a bug somewhere. Die hard here
		 * instead of just returning 0; we could be somewhere in a subprog,
		 * so execution could continue otherwise which we do /not/ want.
		 *
		 * Note, verifier whitelists all opcodes in bpf_opcode_in_insntable().
		 */
		pr_warn("BPF interpreter: unknown opcode %02x (imm: 0x%x)\n",
			insn->code, insn->imm);
		BUG_ON(1);
		return 0;
}
```


### eBPF JIT编译器


eBPF JIT编译器主要是将eBPF程序字节码转换成机器码。由于eBPF指令集与机器指令集存在着一一对应的关系，所以JIT实现起来也比较简单。由于流程过于繁琐，本文不会展开讲解。下面是x86-64平台的eBPF jit编译器实现代码，关键部分会有注释讲解。


#### eBPF指令和x86-64指令映射关系


```c
#define X86_JB  0x72
#define X86_JAE 0x73
#define X86_JE  0x74
#define X86_JNE 0x75
#define X86_JBE 0x76
#define X86_JA  0x77
#define X86_JL  0x7C
#define X86_JGE 0x7D
#define X86_JLE 0x7E
#define X86_JG  0x7F
```

#### eBPF寄存器和x86-64指令映射关系

```c
static const int reg2hex[] = {
	[BPF_REG_0] = 0,  /* RAX */
	[BPF_REG_1] = 7,  /* RDI */
	[BPF_REG_2] = 6,  /* RSI */
	[BPF_REG_3] = 2,  /* RDX */
	[BPF_REG_4] = 1,  /* RCX */
	[BPF_REG_5] = 0,  /* R8  */
	[BPF_REG_6] = 3,  /* RBX callee saved */
	[BPF_REG_7] = 5,  /* R13 callee saved */
	[BPF_REG_8] = 6,  /* R14 callee saved */
	[BPF_REG_9] = 7,  /* R15 callee saved */
	[BPF_REG_FP] = 5, /* RBP readonly */
	[BPF_REG_AX] = 2, /* R10 temp register */
	[AUX_REG] = 3,    /* R11 temp register */
	[X86_REG_R9] = 1, /* R9 register, 6th function argument */
};
```

#### x86-64 栈帧

	

#### 生成 prologue

在计算机程序中，prologue是指程序的开头部分，通常包括一系列指令，用于设置函数调用所需的状态。在函数调用开始之前，程序需要先执行prologue中的指令，以确保函数调用所需的环境和状态已经被正确地构建和初始化。

在x86架构中，prologue通常包括以下指令：

push rbp：将当前函数的基址指针入栈，以保存调用函数之前的rbp值。
mov rbp, rsp：将当前函数的栈顶指针复制到rbp寄存器中，以作为基址指针使用。
sub rsp, n：为当前函数分配一定的栈空间，用于存储局部变量和函数参数。
push ebx/esi/edi：将其他寄存器的值（如ebx、esi、edi）入栈，以保存调用函数之前的值。


```c
static void emit_prologue(u8 **pprog, u32 stack_depth, bool ebpf_from_cbpf,
			  bool tail_call_reachable, bool is_subprog)
{
	u8 *prog = *pprog;

	EMIT_ENDBR();
	memcpy(prog, x86_nops[5], X86_PATCH_SIZE);
	prog += X86_PATCH_SIZE;
	if (!ebpf_from_cbpf) {
		if (tail_call_reachable && !is_subprog)
			EMIT2(0x31, 0xC0); /* xor eax, eax */
		else
			EMIT2(0x66, 0x90); /* nop2 */
	}
	EMIT1(0x55);             /* push rbp */
	EMIT3(0x48, 0x89, 0xE5); /* mov rbp, rsp */

	/* X86_TAIL_CALL_OFFSET is here */
	EMIT_ENDBR();

	/* sub rsp, rounded_stack_depth */
	if (stack_depth)
		EMIT3_off32(0x48, 0x81, 0xEC, round_up(stack_depth, 8));
	if (tail_call_reachable)
		EMIT1(0x50);         /* push rax */
	*pprog = prog;
}
```





#### callee-saved寄存器压栈


x86-64 register R12 is unused

我们知道eBPF的callee-saved寄存器是R6-R9，相应的x86-64的callee-saved寄存器是

RBX：基地址寄存器，保存数组和数据结构的基地址。
RBP：基指针寄存器，保存栈帧的基地址。
R12-R15：通用寄存器，用于存储临时变量和中间结果。

在函数调用过程中，callee-saved和caller-saved是两种常见的寄存器保存策略。

callee-saved寄存器是指在函数调用前，被调用函数需要将其使用的一些寄存器值保存在栈中，以保证在函数返回后能够正确恢复原始的寄存器状态。callee-saved寄存器通常是被调用函数自己使用的，不会影响到调用者函数的寄存器状态。例如，在x86架构中，callee-saved寄存器包括ebx、esi、edi等。

caller-saved寄存器则是指在函数调用前，调用函数需要将其使用的一些寄存器值保存在栈中，以保证在函数返回后能够正确恢复原始的寄存器状态。与callee-saved寄存器不同的是，caller-saved寄存器通常是调用函数（caller）自己使用的，被调用函数（callee）不会修改它们的值。例如，在x86架构中，caller-saved寄存器包括eax、ecx、edx等。

综上所述，callee-saved和caller-saved是函数调用过程中常见的寄存器保存策略，它们的使用取决于函数的具体实现和寄存器的使用情况。

```c
static void push_callee_regs(u8 **pprog, bool *callee_regs_used)
{
	u8 *prog = *pprog;

	if (callee_regs_used[0])
		EMIT1(0x53);         /* push rbx */
	if (callee_regs_used[1])
		EMIT2(0x41, 0x55);   /* push r13 */
	if (callee_regs_used[2])
		EMIT2(0x41, 0x56);   /* push r14 */
	if (callee_regs_used[3])
		EMIT2(0x41, 0x57);   /* push r15 */
	*pprog = prog;
}
```


#### 指令集转换

在设计之初，eBPF的指令集就是尽可能的去贴合机器指令集，所以JIT这里，只需要对每条eBPF指令进行处理，将其转换成相应的机器指令即可。

```c

static int do_jit(struct bpf_prog *bpf_prog, int *addrs, u8 *image, u8 *rw_image,
		  int oldproglen, struct jit_context *ctx, bool jmp_padding)
{
	// ......
	for (i = 1; i <= insn_cnt; i++, insn++) {
		// ......
		switch (insn->code) {
		// ......
		case BPF_JMP | BPF_EXIT:
			if (seen_exit) {
				jmp_offset = ctx->cleanup_addr - addrs[i];
				goto emit_jmp;
			}
			seen_exit = true;
			ctx->cleanup_addr = proglen;
			pop_callee_regs(&prog, callee_regs_used);
			EMIT1(0xC9);         /* leave */
			emit_return(&prog, image + addrs[i - 1] + (prog - temp));
			break;
		}
		// ......
	}
	// ......
}

static void emit_mov_reg(u8 **pprog, bool is64, u32 dst_reg, u32 src_reg)
{
	u8 *prog = *pprog;

	if (is64) {
		/* mov dst, src */
		EMIT_mov(dst_reg, src_reg);
	} else {
		/* mov32 dst, src */
		if (is_ereg(dst_reg) || is_ereg(src_reg))
			EMIT1(add_2mod(0x40, dst_reg, src_reg));
		EMIT2(0x89, add_2reg(0xC0, dst_reg, src_reg));
	}

	*pprog = prog;
}
```

#### callee-saved寄存器出栈

在程序结束时，需要恢复callee-saved寄存器的值，即将callee-saved寄存器出栈。和callee-saved寄存器入栈时一样，需要按照相反的顺序将r15，r14，r13和rbx寄存器弹出栈。

下面是内核do_jit函数代码片段，可以看到当遇到eBPF BPF_EXIT指令时，则恢复callee-saved寄存器，然后是添加leave和ret指令。

leave和生成prologue也是对应关系
其中leave指令的作用相当于执行mov rsp, rbp 和 pop rbp两条指令，即恢复到caller的栈帧。
leave指令的作用相当于执行mov rsp, rbp 和 pop rbp两条指令，即将栈指针rsp设置为栈帧基指针rbp的值，然后将栈帧基指针rbp弹出栈。ret指令的作用是将栈顶元素弹出栈，并将其作为返回地址执行跳转。也就是调用者的下一条指令。

```c
static int do_jit(struct bpf_prog *bpf_prog, int *addrs, u8 *image, u8 *rw_image,
		  int oldproglen, struct jit_context *ctx, bool jmp_padding)
{
	// ......
	for (i = 1; i <= insn_cnt; i++, insn++) {
		// ......
		switch (insn->code) {
		// ......
		case BPF_JMP | BPF_EXIT:
			if (seen_exit) {
				jmp_offset = ctx->cleanup_addr - addrs[i];
				goto emit_jmp;
			}
			seen_exit = true;
			// 标记程序退出位置
			ctx->cleanup_addr = proglen;
			pop_callee_regs(&prog, callee_regs_used);
			EMIT1(0xC9);         /* leave */
			emit_return(&prog, image + addrs[i - 1] + (prog - temp));  /* ret */
			break;
		}
		// ......
	}
	// ......
}
```