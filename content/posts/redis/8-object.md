---
title: "8 Object"
date: 2020-08-12T16:37:27+08:00
description: ""
draft: true
tags: []
categories: []
---


LRU是最近最少使用页面置换算法(Least Recently Used),也就是首先淘汰最长时间未被使用的页面!
LFU是最近最不常用页面置换算法(Least Frequently Used),也就是淘汰一定时期内被访问次数最少的页!


### 基本结构

```c
typedef struct redisObject {
    unsigned type:4;            // 数据类型
    unsigned encoding:4;        // 数据编码方式
    unsigned lru:LRU_BITS;      // lru时间计数
    int refcount;               // 引用计数
    void *ptr;                  // 具体的数据指针
} robj;
```

robj通过type来区分不同的对象类型，对象类型有如下7种：

```c
#define OBJ_STRING 0    // 字符串
#define OBJ_LIST 1      // 列表
#define OBJ_SET 2       // 集合
#define OBJ_ZSET 3      // 有序集合
#define OBJ_HASH 4      // 哈希
#define OBJ_MODULE 5    /* Module object. */
#define OBJ_STREAM 6    /* Stream object. */
```

robj通过encoding来区分不同的数据类型，编码方式有如下11种：

```c
#define OBJ_ENCODING_RAW 0          // 简单动态字符串
#define OBJ_ENCODING_INT 1          // ints
#define OBJ_ENCODING_HT 2           // 字典
#define OBJ_ENCODING_ZIPMAP 3       // 压缩map
#define OBJ_ENCODING_LINKEDLIST 4   // 双端链表
#define OBJ_ENCODING_ZIPLIST 5      // 压缩列表
#define OBJ_ENCODING_INTSET 6       // 整数集合
#define OBJ_ENCODING_SKIPLIST 7     // 跳跃表
#define OBJ_ENCODING_EMBSTR 8       // embstr编码的sds
#define OBJ_ENCODING_QUICKLIST 9    // quicklist
#define OBJ_ENCODING_STREAM 10      // radix tree
```


### 基本方法


### 创建robj

```c
robj *createObject(int type, void *ptr) {
    robj *o = zmalloc(sizeof(*o));
    o->type = type; // redis的对象类型，字符串，哈希表等
    o->encoding = OBJ_ENCODING_RAW;
    o->ptr = ptr;
    o->refcount = 1;    //引用计数的初始值

    /* Set the LRU to the current lruclock (minutes resolution), or
     * alternatively the LFU counter. */
        /* LFU下o->lru的结构如下：
        访问时间     访问频率          
    +-----------------------+
    |     16 bit    | 8 bit |
    +-----------------------+
       如果maxmemory策略是MAXMEMORY_FLAG_LFU，将16bit时间部分设置为当前时间(的低16位)，初始的计数为LFU_INIT_VAL */
    if (server.maxmemory_policy & MAXMEMORY_FLAG_LFU) {
        o->lru = (LFUGetTimeInMinutes()<<8) | LFU_INIT_VAL;
    } else {
        o->lru = LRU_CLOCK();
    }
    return o;
}
```



REDIS_STRING	REDIS_ENCODING_INT	使用整型值实现的字符串对象
REDIS_STRING	REDIS_ENCODING_ EMBSTR	使用embstr编码的简单动态字符串实现的字符串对象
REDIS_STRING	REDIS_ENCODING_ RAW	使用简单动态字符串实现的字符串对象
REDIS_LIST	REDIS_ENCODING_ZIPLIST	使用压缩列表实现的列表对象
REDIS_LIST	REDIS_ENCODING_ LINKEDLIST	使用双向链表实现的列表对象
REDIS_HASH	REDIS_ENCODING_ZIPLIST	使用压缩列表实现的哈希对象
REDIS_HASH	REDIS_ENCODING_HT	使用字典实现的哈希对象
REDIS_SET	REDIS_ENCODING_INTSET	使用整数集合实现的集合对象
REDIS_SET	REDIS_ENCODING_HT	使用字典实现的集合对象
REDIS_ZSET	REDIS_ENCODING_ZIPLIST	使用压缩列表实现的有序集合对象
REDIS_ZSET	REDIS_ENCODING_SKIPLIST	使用跳跃表和字典实现的有序集合对象
