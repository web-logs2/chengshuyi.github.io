---
title: "lwip arp源码阅读笔记"
date: 2020-04-28T16:05:01+08:00
description: ""
draft: true
tags: [计算机网络,lwip]
categories: [计算机网络,lwip]
---



本文主要介绍了lwip。



| 层次       | 协议                    |      |
| ---------- | ----------------------- | ---- |
| 应用层     | Telnet、FTP、SMTP、SNMP |      |
| 传输层     | TCP、UDP                |      |
| 网络层     | ip、ICMP、IGMP          |      |
| 数据链路层 |                         |      |





```c
//mac层协议头部
struct eth_hdr {
#if ETH_PAD_SIZE
  PACK_STRUCT_FLD_8(u8_t padding[ETH_PAD_SIZE]);
#endif
  PACK_STRUCT_FLD_S(struct eth_addr dest);
  PACK_STRUCT_FLD_S(struct eth_addr src);
  PACK_STRUCT_FIELD(u16_t type);
} 
//arp协议头部
struct etharp_hdr {
  PACK_STRUCT_FIELD(u16_t hwtype);
  PACK_STRUCT_FIELD(u16_t proto);
  PACK_STRUCT_FLD_8(u8_t  hwlen);
  PACK_STRUCT_FLD_8(u8_t  protolen);
  PACK_STRUCT_FIELD(u16_t opcode);
  PACK_STRUCT_FLD_S(struct eth_addr shwaddr);
  PACK_STRUCT_FLD_S(struct ip4_addr_wordaligned sipaddr);
  PACK_STRUCT_FLD_S(struct eth_addr dhwaddr);
  PACK_STRUCT_FLD_S(struct ip4_addr_wordaligned dipaddr);
}
```

