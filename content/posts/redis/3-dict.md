---
title: "redis源码分析三：dict数据结构"
date: 2020-08-12T16:35:16+08:00
description: ""
draft: false
tags: [redis源码分析]
categories: [redis源码分析]
---



dictEntry 

dictht

dict



dictType

dictIterator



redis提供了如下几个dict操作：

* 创建一个dict：`dictCreate`

* dict扩张，设置rehash相关参数：`dictExpand`

* 向dict添加一个{key,vale}，不支持覆盖：`dictAdd`
* 向dict添加一个{key,vale}，支持覆盖：`dictReplace`
* 从dict删除一个key：`dictDelete`
* dict rehash操作：`dictRehash`
* 

`dictDelete`

`dictFind`



### 基本结构



### 哈希算法

redis采用的哈希算法默认是siphash算法。

### 创建dict



```c
dict *dictCreate(dictType *type,
        void *privDataPtr)
{
    dict *d = zmalloc(sizeof(*d));  // 1. 分配内存：dict
    _dictInit(d,type,privDataPtr);
    return d;
}
int _dictInit(dict *d, dictType *type,
        void *privDataPtr)
{
    _dictReset(&d->ht[0]);          // 2. 初始化ht[0]
    _dictReset(&d->ht[1]);          // 3. 初始化ht[1]
    d->type = type;                 // 4. dictType
    d->privdata = privDataPtr;      // 5. 私有数据
    d->rehashidx = -1;              // 6. 未进行rehash
    d->iterators = 0;               // 7. 迭代器的个数
    return DICT_OK;
}
static void _dictReset(dictht *ht)
{
    ht->table = NULL;
    ht->size = 0;
    ht->sizemask = 0;
    ht->used = 0;
}
```

### 添加键

添加键需要重点关注以下几个关键点：

* 执行添加键并且安全迭代器个数为0的话可以触发渐进式rehash；
* `_dictKeyIndex`根据key获取index，其中必须遍历ht[0]和ht[1]。因为在rehash过程中，对键的修改操作都是直接操作到ht[1]上；
* 如果在rehash过程中，则直接将键插入到ht[1]上；

```c
/* Add an element to the target hash table */
int dictAdd(dict *d, void *key, void *val)
{
    dictEntry *entry = dictAddRaw(d,key,NULL);
    if (!entry) return DICT_ERR;
    dictSetVal(d, entry, val);			// 8. 拷贝复制（如果自定义拷贝函数存在的话）或者直接复制
    return DICT_OK;
}
dictEntry *dictAddRaw(dict *d, void *key, dictEntry **existing)
{
    long index;
    dictEntry *entry;
    dictht *ht;
    if (dictIsRehashing(d)) _dictRehashStep(d);	// 1. 渐进式rehash，执行单步rehash（但是得确保当前安全迭代器的个数为0）
    // 2. 获取index = HashFunction(key)
    if ((index = _dictKeyIndex(d, key, dictHashKey(d,key), existing)) == -1)
        return NULL;
	// 3. 根据是否在进行rehash，选择哈希表
    ht = dictIsRehashing(d) ? &d->ht[1] : &d->ht[0];
    entry = zmalloc(sizeof(*entry));    // 4. 分配表项
    entry->next = ht->table[index];     // 5. 头插法
    ht->table[index] = entry;
    ht->used++;                         // 6. 表项个数
    dictSetKey(d, entry, key);			// 7. 拷贝复制（如果自定义拷贝函数存在的话）或者直接复制
    return entry;
}
```

### 删除键



### 哈希表扩张/收缩







### 渐进式rehash





