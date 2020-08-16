---
title: "redis源码分析六：intset整数集合"
date: 2020-08-12T16:36:41+08:00
description: ""
draft: false
tags: [redis源码分析]
categories: [redis源码分析]
---


### 基本结构

```c
typedef struct intset {
    uint32_t encoding;  // 编码方式:16 32 64三种
    uint32_t length;    // 集合包含的元素数量
    int8_t contents[];  // 保存元素的数组
} intset;
```

### 基本方法

* intsetNew：创建一个空的整数集合
* intsetAdd：向整数集合添加元素；
* intsetRemove：从整数集合删除元素；
* intsetFind
* intsetRandom
* intsetGet
* intsetLen
* intsetBlobLen

### 创建整数集合

```c
intset *intsetNew(void) {
    intset *is = zmalloc(sizeof(intset));           // 1. 为整数集合结构分配空间
    is->encoding = intrev32ifbe(INTSET_ENC_INT16);  // 2. 设置初始编码
    is->length = 0;                                 // 3. 初始化元素数量
    return is;
}
```


### 添加元素

整数集合一个比较大的特点就是能够根据插入的值的大小动态的选择编码方式。

```c
intset *intsetAdd(intset *is, int64_t value, uint8_t *success) {
    uint8_t valenc = _intsetValueEncoding(value);               // 1. 根据值的大小选择不同的编码方式
    uint32_t pos;
    if (success) *success = 1;
    if (valenc > intrev32ifbe(is->encoding)) {                  // 2. 编码长度升级 
        return intsetUpgradeAndAdd(is,value);
    } else {
        if (intsetSearch(is,value,&pos)) {                      // 3. 找到其应该插入的位置
            if (success) *success = 0;
            return is;
        }
        is = intsetResize(is,intrev32ifbe(is->length)+1);       // 4. 为 value 在集合中分配空间
        if (pos < intrev32ifbe(is->length)) intsetMoveTail(is,pos,pos+1); // 5. 如果新元素不是被添加到底层数组的末尾那么需要对现有元素的数据进行移动，空出 pos 上的位置，用于设置新值
    }
    _intsetSet(is,pos,value);                                   // 6. 将新值设置到底层数组的指定位置中
    is->length = intrev32ifbe(intrev32ifbe(is->length)+1);
    return is;
}
```

### 删除元素



```c
intset *intsetRemove(intset *is, int64_t value, int *success) {
    uint8_t valenc = _intsetValueEncoding(value);            // 1. 计算 value 的编码方式
    uint32_t pos;
    if (success) *success = 0;
    if (valenc <= intrev32ifbe(is->encoding) && intsetSearch(is,value,&pos)) {
        uint32_t len = intrev32ifbe(is->length);
        if (success) *success = 1;
        if (pos < (len-1)) intsetMoveTail(is,pos+1,pos);    // 2. 如果 value 不是位于数组的末尾，那么需要对原本位于 value 之后的元素进行移动
        is = intsetResize(is,len-1);                        // 3. 缩小数组的大小，移除被删除元素占用的空间
        is->length = intrev32ifbe(len-1);                   // 4. 更新长度
    }
    return is;
}
```

### 源码文件

1. `src/intset.h`
2. `src/intset.c`