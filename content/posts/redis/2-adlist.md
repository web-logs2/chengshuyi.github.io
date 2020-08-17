---
title: "redis源码分析二：adlist数据结构"
date: 2020-08-12T16:35:08+08:00
description: ""
draft: false
tags: [redis源码分析]
categories: [redis源码分析]
---


redis实现了双向链表，其名称是adlist（A generic Doubly linked list implementation）。adlist提供了以下几种操作方法：

* 创建双向链表
* 释放双向链表：释放节点、释放节点值、释放双向链表
* 头插法
* 尾插法
* 删除节点
* 迭代器：头部迭代或者尾部迭代

本文主要讲解了adlist数据结构、创建双向链表和头插法。其余方法不作介绍。

### adlist数据结构

见注释：

```c
typedef struct listNode {
    struct listNode *prev;              // 前驱节点
    struct listNode *next;              // 后驱节点
    void *value;                        // 节点值
} listNode;

typedef struct listIter {               
    listNode *next;                     // 迭代器：方便遍历
    int direction;                      // 指明方向，从头到尾还是从尾到头
} listIter;

typedef struct list {                   // 双向链表结构体，注意同节点结构体区分开来
    listNode *head;                     // 头部节点指针
    listNode *tail;                     // 尾部节点指针
    void *(*dup)(void *ptr);            // 自定义复制函数
    void (*free)(void *ptr);            // 自定义释放函数
    int (*match)(void *ptr, void *key); // 自定义匹配函数，用于查找
    unsigned long len;                  // 链表长度
} list;

```

### 创建双向链表

双向链表创建只需要分配双向链表结构体即可，具体见注释：

```c
list *listCreate(void)
{
    struct list *list;

    if ((list = zmalloc(sizeof(*list))) == NULL)    // 1. 分配内存：struct list
        return NULL;
    list->head = list->tail = NULL;                 // 2. 双向链表置空
    list->len = 0;
    list->dup = NULL;
    list->free = NULL;
    list->match = NULL;
    return list;
}
```

### 双向链表头插

见注释：

```c
list *listAddNodeHead(list *list, void *value)
{
    listNode *node;
    if ((node = zmalloc(sizeof(*node))) == NULL)    // 1. 分配一个新的节点listNode
        return NULL;        
    node->value = value;                            // 2. 保存数据到该节点                        
    if (list->len == 0) {                           // 3. list为空，则头尾都指向新节点
        list->head = list->tail = node;
        node->prev = node->next = NULL;
    } else {                                        // 4. list不为空，执行头插
        node->prev = NULL;
        node->next = list->head;
        list->head->prev = node;
        list->head = node;
    }
    list->len++;                                    // 5. 长度+1
    return list;
}
```

### 源码文件：

1. `src/adlist.h`
2. `src/adlist.c`

