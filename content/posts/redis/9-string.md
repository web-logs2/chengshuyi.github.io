---
title: "redis源码分析九：字符串键"
date: 2020-08-12T16:37:35+08:00
description: ""
draft: true
tags: [redis源码分析]
categories: [redis源码分析]
---





### 基本命令

> 下表来自：https://blog.csdn.net/harleylau/article/details/80263144

| 命令                                      | 命令描述                                   |
| ----------------------------------------- | ------------------------------------------ |
| SET key value \[ex 秒数\]\[px 毫秒数\][nx/xx] | 设置指定key的值                            |
| GET key                                   | 获取指定key的值                            |
| APPEND key value                          | 将value追加到指定key的值末尾               |
| INCRBY key increment                      | 将指定key的值加上增量increment             |
| DECRBY key decrement                      | 将指定key的值减去增量decrement             |
| STRLEN key                                | 返回指定key的值长度                        |
| SETRANGE key offset value                 | 将value覆写到指定key的值上，从offset位开始 |
| GETRANGE key start end                    | 获取指定key中字符串的子串[start,end]       |
| MSET key value [key value …]              | 一次设定多个key的值                        |
| MGET key1 [key2..]                        | 一次获取多个key的值                        |