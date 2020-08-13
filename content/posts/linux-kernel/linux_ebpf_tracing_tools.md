---
title: "Linux_ebpf_tracing_tools"
date: 2020-08-09T20:34:22+08:00
description: ""
draft: true
tags: []
categories: []
---

> 译自：http://www.brendangregg.com/ebpf.html

本文介绍了性能分析工具的示例，这些示例使用了BPF（Berkerly Packet filter）改进版本，这些改进功能已添加到Linux 4.x系列内核中，从而使BPF可以做的不仅仅是过滤数据包。这些改进功能允许**自定义分析程序**运行在Linux提供动态跟踪，静态跟踪和事件机制上。

BPF跟踪的主要前端和推荐前端是BCC和bpftrace：复杂工具和守护程序的BCC以及单行代码和简短脚本的bpftrace。如果您正在寻找要运行的工具，请尝试BCC，然后尝试bpftrace。如果要自己编程，请从bpftrace开始，并且仅在需要时使用BCC。我已经将许多DTraceToolkit脚本移植到了BCC和bpftrace，它们的存储库在它们之间提供了100多种工具。我还为即将出版的书《BPF性能工具：Linux系统和应用程序可观察性》开发了100多本书。


funccount
trace
argdist