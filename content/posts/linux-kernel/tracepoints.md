---
title: "Tracepoints"
date: 2020-05-13T12:48:11+08:00
description: ""
draft: true
tags: []
categories: []
---


如何插入tracepoint
将其同probe函数链接

tracepoint是一个Hook函数，能够调用Probe函数
当

#define DECLARE_TRACE(name, proto, args)				\
	__DECLARE_TRACE(name, PARAMS(proto), PARAMS(args),		\
			cpu_online(raw_smp_processor_id()),		\
			PARAMS(void *__data, proto),			\
			PARAMS(__data, args))

#define __DECLARE_TRACE(name, proto, args, cond, data_proto, data_args) \
	static inline void trace_##name(proto)				\
	{ }								\
	static inline void trace_##name##_rcuidle(proto)		\
	{ }								\
	static inline int						\
	register_trace_##name(void (*probe)(data_proto),		\
			      void *data)				\
	{								\
		return -ENOSYS;						\
	}								\
	static inline int						\
	unregister_trace_##name(void (*probe)(data_proto),		\
				void *data)				\
	{								\
		return -ENOSYS;						\
	}								\
	static inline void check_trace_callback_type_##name(void (*cb)(data_proto)) \
	{								\
	}								\
	static inline bool						\
	trace_##name##_enabled(void)					\
	{								\
		return false;						\
	}


#define DEFINE_TRACE_FN(name, reg, unreg)				 \
	static const char __tpstrtab_##name[]				 \
	__attribute__((section("__tracepoints_strings"))) = #name;	 \
	struct tracepoint __tracepoint_##name				 \
	__attribute__((section("__tracepoints"), used)) =		 \
		{ __tpstrtab_##name, STATIC_KEY_INIT_FALSE, reg, unreg, NULL };\
	__TRACEPOINT_ENTRY(name);

#define DEFINE_TRACE(name)						\
	DEFINE_TRACE_FN(name, NULL, NULL);