---
title: "Malloc"
date: 2020-08-09T11:04:16+08:00
description: ""
draft: true
tags: []
categories: []
---


```c
#include "unistd.h"
#include "stdio.h"
#include "stdlib.h"

int main(){
    printf("%p\n",sbrk(0));
    int *a = (int*) malloc(4*1024);
    printf("%p\n",sbrk(0));
    int *b = (int*) malloc(4*1024);
    printf("%p\n",sbrk(0));
    return 0;
}
0x55676ebb0000
0x55676ebd1000
0x55676ebd1000
```