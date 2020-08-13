---
title: "redis源码分析四：跳跃表"
date: 2020-08-12T16:35:33+08:00
description: ""
draft: false
tags: [redis源码分析]
categories: [redis源码分析]
---

创建 zslCreate
插入 zslInsert
删除 zslDelete

### 跳跃表基础知识

> 摘自：https://blog.csdn.net/universe_ant/article/details/51134020

跳跃表(skiplist)是一种有序数据结构，它通过在**每个节点中维持多个指向其他节点的指针（也就是level）**，从而达到快速访问节点的目的。跳跃表支持平均O(logN)、最坏O(N)复杂度的节点查找，还可以通过顺序性操作来批量处理节点。在大部分情况下，跳跃表的效率可以和平衡树相媲美，并且因为跳跃表的实现比平衡树要来得更为简单，所以有不少程序都使用跳跃表来代替平衡树。Redis使用跳跃表作为有序集合键的底层实现之一，如果一个有序集合包含的元素数量比较多，又或者有序集合中元素的成员(member)是比较长的字符串时，Redis就会使用跳跃表来作为有序集合键的底层实现。和链表、字典等数据结构被广泛地应用在Redis内部不同，Redis只在两个地方用到了跳跃表，一个是实现有序集合键，另一个是在集群节点中用作内部数据结构，除此之外，跳跃表在Redis里面没有其他用途。

![](https://gitee.com/chengshuyi/scripts/raw/master/img/20200813133552.png)

### 跳跃表数据结构

跳跃表核心数据结构有两个：

* zskiplistNode：同一个节点在每层的score和sds是一致的；backward指针用于从尾到头遍历，跨度恒为1；每一层会变化的则放在zskiplistLevel结构中；
* zskiplist：保存跳跃表节点的相关信息；

```c
typedef struct zskiplistNode {
    sds ele;                                // sds
    double score;                           // 得分，按分数从小到大排序
    struct zskiplistNode *backward;         // 从尾到头遍历
    struct zskiplistLevel {                 
        struct zskiplistNode *forward;      // 从头到尾遍历
        unsigned long span;                 // 跨度
    } level[];                              // 最大32层
} zskiplistNode;

typedef struct zskiplist {
    struct zskiplistNode *header, *tail;    // 头尾指针，不存实际的数据
    unsigned long length;                   // 节点数量
    int level;                              // level最大值（不包括头节点）
} zskiplist;
```

### 跳跃表的创建

见注释：

```c
zskiplistNode *zslCreateNode(int level, double score, sds ele) {
    zskiplistNode *zn =
        zmalloc(sizeof(*zn)+level*sizeof(struct zskiplistLevel));
    zn->score = score;
    zn->ele = ele;
    return zn;
}
zskiplist *zslCreate(void) {
    int j;
    zskiplist *zsl;

    zsl = zmalloc(sizeof(*zsl));                            
    zsl->level = 1;                                         // 1. zsl的初始level是1
    zsl->length = 0;
    zsl->header = zslCreateNode(ZSKIPLIST_MAXLEVEL,0,NULL); // 2. 头节点的level = 32，但是不计入zsl.level中
    for (j = 0; j < ZSKIPLIST_MAXLEVEL; j++) {
        zsl->header->level[j].forward = NULL;               // 3. 头节点每一层的前进指针置空、跨度为0
        zsl->header->level[j].span = 0;
    }
    zsl->header->backward = NULL;
    zsl->tail = NULL;                                       // 4. 
    return zsl;
}
```

### 跳跃表的插入

需要关注的几个点是：

* rank数组：表示每一层遍历到截止点的跨度（经历的节点数量）；
* update数组：插入新节点后会受到影响的节点（每一层都会有一个）；
* 随机生成level：
* 有了rank和update后，后面的操作可以看成每一个层级对双向链表的操作；

具体见注释：

```c
zskiplistNode *zslInsert(zskiplist *zsl, double score, sds ele) {
    zskiplistNode *update[ZSKIPLIST_MAXLEVEL], *x;
    unsigned int rank[ZSKIPLIST_MAXLEVEL];
    int i, level;

    serverAssert(!isnan(score));
    x = zsl->header;
    for (i = zsl->level-1; i >= 0; i--) {                               // 1. 从最高level往下遍历
        rank[i] = i == (zsl->level-1) ? 0 : rank[i+1];                  // 2. rank表示当前新插入的节点在每一层的跨度
        while (x->level[i].forward &&
                (x->level[i].forward->score < score ||                  // 3. 比对分值，forward节点的score小于待插入的score，则继续遍历；
                    (x->level[i].forward->score == score &&
                    sdscmp(x->level[i].forward->ele,ele) < 0)))         // 4. score一致时，比对成员，如果forward节点的sds小于待插入的sds，则继续遍历；
        {
            rank[i] += x->level[i].span;                                // 5. 记录沿途跨越了多少个节点
            x = x->level[i].forward;                                    // 6. 移动至下一指针
        }
        update[i] = x;                                                  // 7. 记录当前层i遍历到的截止点（新节点会被插入在该截止点后面）；
    }
    // zslInsert() 的调用者会确保同分值且同成员的元素不会出现，所以这里不需要进一步进行检查，可以直接创建新元素。
    level = zslRandomLevel();                                           // 8. 获取一个随机值作为新节点的层数
    if (level > zsl->level) {                                           // 9. 如果新节点的层数比表中其他节点的层数都要大
        for (i = zsl->level; i < level; i++) {
            rank[i] = 0;                                                // 10.
            update[i] = zsl->header;                                    // 11. 
            update[i]->level[i].span = zsl->length;                     // 12. 将来也指向新节点
        }
        zsl->level = level;                                             // 13. 更新zsl的最大层数
    }
    
    x = zslCreateNode(level,score,ele);                                 // 14. 创建新节点
    for (i = 0; i < level; i++) {
        x->level[i].forward = update[i]->level[i].forward;              // 15. 设置新节点的forward指针
        update[i]->level[i].forward = x;                                // 16. 将各层的截止点的forward指针指向新节点
        x->level[i].span = update[i]->level[i].span - (rank[0] - rank[i]);  //17.  计算新节点在各层跨越的节点数量
        update[i]->level[i].span = (rank[0] - rank[i]) + 1;             // 18. 更新新节点插入之后，截止点的span值
    }
    for (i = level; i < zsl->level; i++) {                              
        update[i]->level[i].span++;                                     // 19. 新层的span值也需要增一，编号11：update[i]对应着头部，也就是头部节点到新节点的跨度
    }

    x->backward = (update[0] == zsl->header) ? NULL : update[0];        // 20. 设置新节点的后退指针
    if (x->level[0].forward)
        x->level[0].forward->backward = x;
    else
        zsl->tail = x;
    zsl->length++;                                                      // 21. 跳跃表的节点计数增一
    return x;
}
```

### 跳跃表的删除

类似于插入，暂不分析。

### 源码文件

1. `src/server.h`
2. `src/t_zset.c`