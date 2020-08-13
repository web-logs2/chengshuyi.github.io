---
title: "redis源码分析三：dict数据结构"
date: 2020-08-12T16:35:16+08:00
description: ""
draft: false
tags: [redis源码分析]
categories: [redis源码分析]
---



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


```c
typedef struct dictEntry {
    void *key;              // 哈希表的键
    union {                 // 联合体，键值可以是指针、uint64、int64_t或者double
        void *val;
        uint64_t u64;
        int64_t s64;
        double d;
    } v;
    struct dictEntry *next; // 链地址法解决哈希冲突
} dictEntry;

typedef struct dictType {
    uint64_t (*hashFunction)(const void *key);          // 自定义哈希算法
    void *(*keyDup)(void *privdata, const void *key);   // key的自定义拷贝函数 
    void *(*valDup)(void *privdata, const void *obj);   // val的自定义拷贝函数
    int (*keyCompare)(void *privdata, const void *key1, const void *key2); // 比较算法
    void (*keyDestructor)(void *privdata, void *key);   // key的释放
    void (*valDestructor)(void *privdata, void *obj);   // val的释放
} dictType;

/* This is our hash table structure. Every dictionary has two of this as we
 * implement incremental rehashing, for the old to the new table. */
typedef struct dictht {
    dictEntry **table;          // 哈希表
    unsigned long size;         // 大小 
    unsigned long sizemask;     // 哈希表大小的掩码
    unsigned long used;         // 当前哈希表存在dictEntry的个数
} dictht;

typedef struct dict {
    dictType *type;             // 提供dict的自定义函数
    void *privdata;             // 私有数据
    dictht ht[2];               // ht[0]作为原始哈希表,ht[1]作为rehash
    long rehashidx;             // rehash标志位，-1没有rehash
    unsigned long iterators;    // 安全迭代器的个数，防止rehash和修改操作冲突
} dict;
```


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

类似于插入，暂不分析。

### 更新键

类似于插入，暂不分析。

### rehash

redis rehash分为两种：

* 渐进式rehash：将rehash操作分布在每个增加、删除、修改操作上；
* 集中式rehash：一次性rehash完成；

```c
int dictRehash(dict *d, int n) {
    int empty_visits = n*10;                                    // 1. 访问到empty_visits个空桶的话直接返回
    if (!dictIsRehashing(d)) return 0;
    while(n-- && d->ht[0].used != 0) {                          // 2. 最多访问n个有效桶
        dictEntry *de, *nextde;
        while(d->ht[0].table[d->rehashidx] == NULL) {           // 3. 找到第一个不为空的桶
            d->rehashidx++;
            if (--empty_visits == 0) return 1;
        }
        de = d->ht[0].table[d->rehashidx];
        while(de) {                                             // 4. 遍历桶内节点
            uint64_t h;
            nextde = de->next;
            h = dictHashKey(d, de->key) & d->ht[1].sizemask;    // 5. 获取该key在ht[1]上的索引，由于ht[1]比ht[0]的尺寸大或小，因此key对应的index可能会有所不同
            de->next = d->ht[1].table[h];                       // 6. 执行头插法，将数据插入到ht[1]
            d->ht[1].table[h] = de;
            d->ht[0].used--;
            d->ht[1].used++;
            de = nextde;
        }
        d->ht[0].table[d->rehashidx] = NULL;                    // 7. 清空ht[0]中对应的桶
        d->rehashidx++;
    }

    if (d->ht[0].used == 0) {                                   // 8. 检查是否已经完成了rehash，恢复相应的标志位，把ht[1]给ht[0]
        zfree(d->ht[0].table);                                  
        d->ht[0] = d->ht[1];
        _dictReset(&d->ht[1]);
        d->rehashidx = -1;
        return 0;
    }
    return 1;
}
```

### 源码文件

1. `src/dict.c`
2. `src/dict.h`


