


https://lore.kernel.org/bpf/20210325015124.1543397-1-kafai@fb.com/t/#m6e241d57e618821aa3ba6235054d551df200e02d

BPF_PSEUDO_KFUNC_CALL

https://docs.kernel.org/bpf/kfuncs.html




## BTF_ids

<!-- https://lore.kernel.org/bpf/20200711215329.41165-4-jolsa@kernel.org -->

The .BTF_ids section encodes BTF ID values that are used within the kernel.

This section is created during the kernel compilation with the help of
macros defined in ``include/linux/btf_ids.h`` header file. Kernel code can
use them to create lists and sets (sorted lists) of BTF ID values.

The ``BTF_ID_LIST`` and ``BTF_ID`` macros define unsorted list of BTF ID values,
with following syntax::

  BTF_ID_LIST(list)
  BTF_ID(type1, name1)
  BTF_ID(type2, name2)

resulting in following layout in .BTF_ids section::

  __BTF_ID__type1__name1__1:
  .zero 4
  __BTF_ID__type2__name2__2:
  .zero 4

The ``u32 list[];`` variable is defined to access the list.

The ``BTF_ID_UNUSED`` macro defines 4 zero bytes. It's used when we
want to define unused entry in BTF_ID_LIST, like::

      BTF_ID_LIST(bpf_skb_output_btf_ids)
      BTF_ID(struct, sk_buff)
      BTF_ID_UNUSED
      BTF_ID(struct, task_struct)

The ``BTF_SET_START/END`` macros pair defines sorted list of BTF ID values
and their count, with following syntax::

  BTF_SET_START(set)
  BTF_ID(type1, name1)
  BTF_ID(type2, name2)
  BTF_SET_END(set)

resulting in following layout in .BTF_ids section::

  __BTF_ID__set__set:
  .zero 4
  __BTF_ID__type1__name1__3:
  .zero 4
  __BTF_ID__type2__name2__4:
  .zero 4

The ``struct btf_id_set set;`` variable is defined to access the list.

The ``typeX`` name can be one of following::

   struct, union, typedef, func

and is used as a filter when resolving the BTF ID value.

All the BTF ID lists and sets are compiled in the .BTF_ids section and
resolved during the linking phase of kernel build by ``resolve_btfids`` tool.

## 往内核添加kfunc

往内核添加kfunc有两种方案，一种是通过wrapper的方式，另外一种是

需要注意的是，给内核添加kfunc函数需要特别注意，因为不当的使用会导致内核崩溃，所以需要限制调用kfunc函数的上下文，以确保安全



### 创建 wrapper


__sz 标记，标记内存大小

```c
__bpf_kfunc void bpf_memzero(void *mem, int mem__sz)
{
...
}
```
内核会将第一个参数看成PTR_TO_MEM，第二个参数大小




### 通过BTF_ID标记

__bpf_kfunc 避免编译器将该函数编译成inline的了，或者被优化掉，因为没有任何使用

`KF_ACQUIRE`



### 注册添加的kfunc

```c
BTF_SET8_START(bpf_task_set)
BTF_ID_FLAGS(func, bpf_get_task_pid, KF_ACQUIRE | KF_RET_NULL)
BTF_ID_FLAGS(func, bpf_put_pid, KF_RELEASE)
BTF_SET8_END(bpf_task_set)

static const struct btf_kfunc_id_set bpf_task_kfunc_set = {
        .owner = THIS_MODULE,
        .set   = &bpf_task_set,
};

static int init_subsystem(void)
{
        return register_btf_kfunc_id_set(BPF_PROG_TYPE_TRACING, &bpf_task_kfunc_set);
}
late_initcall(init_subsystem);
```