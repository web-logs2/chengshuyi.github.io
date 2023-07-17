---
title: "bpftrace: Int"
date: 2023-05-12T16:01:07+08:00
description: ""
draft: true
tags: [eBPF,bpftrace]
categories: [eBPF,bpftrace]
---







目前只有kfunc/kretfunc/uprobe可以使用args

iter可以使用ctx


## iter

```
iter:task_file { printf("%s:%d\n", ctx->task->comm, ctx->task->pid); }
```






```c
void CodegenLLVM::visit(Builtin &builtin)
// ...
  else if (builtin.ident == "args" || builtin.ident == "ctx")
  {
    // ctx is undocumented builtin: for debugging
    // ctx_ is casted to int for arithmetic operation
    // it will be casted to a pointer when loading
    expr_ = b_.CreatePtrToInt(ctx_, b_.getInt64Ty());
  }
```

```c
void SemanticAnalyser::visit(Builtin &builtin)
{
  if (builtin.ident == "ctx")
{
```

## kfunc


```
kfunc:tcp_sendmsg {printf("%d\n", args->sk->sk_sndbuf);}
```








ir 解析



`Value *CreateGEP(Type *Ty, Value *Ptr, Value *Idx, const Twine &Name = "")`

`Value *CreateGEP(Type *Ty, Value *Ptr, ArrayRef<Value *> IdxList, const Twine &Name = "")`

```c
// 入参 uint64_t str2_size;
// 入参 Value *str2
AllocaInst *store = CreateAllocaBPF(getInt1Ty(), "strcmp.result");

auto *ptr_r = CreateGEP(ArrayType::get(getInt8Ty(), str2_size), str2, { getInt32(0), getInt32(i) });
                              
Value *r = CreateLoad(getInt8Ty(), ptr_r);
 CreateStore(getInt1(inverse), store);
```


```c

Value *ptr = CreateGEP(ArrayType::get(getInt8Ty(), ))

```