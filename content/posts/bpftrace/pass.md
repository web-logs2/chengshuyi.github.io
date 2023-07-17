---
title: "bpftrace 编译器 pass"
date: 2023-05-12T16:01:07+08:00
description: ""
draft: true
tags: [eBPF,bpftrace]
categories: [eBPF,bpftrace]
---


bpftrace 目前有`field`、`semantic`、`resource`、`counter`、`portablility`、等几个pass


## field

主要是用于field access解析，解析函数入参的：

1. builtin：args retval ctx   解析程序入参类型，获取其sized type
2. FieldAccess：获取成员类型


好像只是为了加入bpftrace_.btf_set_

## semantic




## counter



## resource

