---
title: "redis源码分析一：sds数据结构"
date: 2020-08-12T16:32:50+08:00
description: ""
draft: false
tags: [redis源码分析]
categories: [redis源码分析]
---

本文主要介绍了redis的字符串。

### sds和sds头部结构体

sds（simple dynamic string）是redis管理字符串的一种数据结构。sds结构体是`char *`类型，指向sds头部结构体的buf。sds头部有五种不同的结构：

```c
struct sdshdr5 {
    unsigned char flags;        // 低三位表示SDS类型，高五位存储buf的长度
    char buf[];
};
struct sdshdr8 {
    uint8_t len;                // buffer的长度
    uint8_t alloc;              // 字符串的实际长度，不包含头部和终止符
    unsigned char flags;        // 低三位表示SDS类型
    char buf[];
};
struct sdshdr16 {
    uint16_t len;               // buffer的长度
    uint16_t alloc;             // 字符串的实际长度，不包含头部和终止符
    unsigned char flags;        // 低三位表示SDS类型
    char buf[];
};
struct sdshdr32 {
    uint32_t len;               // buffer的长度
    uint32_t alloc;             // 字符串的实际长度，不包含头部和终止符
    unsigned char flags;        // 低三位表示SDS类型
    char buf[];
};
struct sdshdr64 {
    uint64_t len;               // buffer的长度
    uint64_t alloc;             // 字符串的实际长度，不包含头部和终止符
    unsigned char flags;        // 低三位表示SDS类型
    char buf[];
};
```

### 创建sds

`sdsnewlen`可以创建一个sds字符串，传入参数是：

* init: 待转换成sds结构体的字符串首地址指针；
* init_len：字符串长度；

```c
sds sdsnewlen(const void *init, size_t initlen) {
    void *sh;
    sds s;
    char type = sdsReqType(initlen);        // 1. 根据初始长度选择合适的sds类型
    if (type == SDS_TYPE_5 && initlen == 0) // 2. 尽量不使用SDS_TYPE_5
        type = SDS_TYPE_8;
    int hdrlen = sdsHdrSize(type);          // 3. 根据sds类型获取sds头部长度
    unsigned char *fp;
    sh = s_malloc(hdrlen+initlen+1);        // 4. 分配内存，后面的+1是'\0'所占空间
    if (init==SDS_NOINIT)
        init = NULL;
    else if (!init)
        memset(sh, 0, hdrlen+initlen+1);    // 5. sds结构体清0
    s = (char*)sh+hdrlen;                   // 6. 指向buf的指针
    fp = ((unsigned char*)s)-1;             // 7. 指向flags的指针
    switch(type) {
        ......
        case SDS_TYPE_8: {
            SDS_HDR_VAR(8,s);               // 8. 根据buf的指针计算出sds头部的首地址，宏定义展开：struct sdshdr8 *sh = (void *)((s)-sizeof(struct sdshdr8))
            sh->len = initlen;       
            sh->alloc = initlen;
            *fp = type;
            break;
        }
        .......
    }
    if (initlen && init)
        memcpy(s, init, initlen);           // 9. 将数据拷贝到sds的buf里
    s[initlen] = '\0';                      // 10. 插入终止符，方便调用printf函数
    return s;                               // 11. 返回的sds是指向buf的指针，可以通过SDS_HDR_VAR得到sds头部的首地址
}
```

### 基本语法：

* char buf[]不占用内存空间，也就是sizeof(struct sdshdr64)等于24，由于对齐的原因flags被扩充到64bit；
* char_arry[-1]取的是当前首地址-1的位置数据；

### 源码文件：

1. `src/sds.h`
2. `src/sds.c`
