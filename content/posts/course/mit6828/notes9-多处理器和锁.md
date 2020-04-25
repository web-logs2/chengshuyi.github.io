---
title: "Notes9 多处理器和锁"
date: 2020-04-25T13:36:18+08:00
description: ""
draft: true
tags: []
categories: []
---

Problem: deadlock
  notice rename() held two locks
  what if:
    core A              core B
    rename(d1/x, d2/y)  rename(d2/a, d1/b)
      lock d1             lock d2
      lock d2 ...         lock d1 ...
  solution:
    programmer works out an order for all locks
    all code must acquire locks in that order
    i.e. predict locks, sort, acquire -- complex!

Locks versus modularity
  locks make it hard to hide details inside modules
  to avoid deadlock, I need to know locks acquired by functions I call
  and I may need to acquire them before calling, even if I don't use them
  i.e. locks are often not the private business of individual modules

How to think about where to place locks?
  here's a simple plan for new code
  1. write module to be correct under serial execution
     i.e. assuming one CPU, one thread
     insert(){ e->next = l; l = e; }
     but not correct if executed in parallel
  2. add locks to FORCE serial execution
     since acquire/release allows execution by only one CPU at a time
    the point:

    it's easier for programmers to reason about serial code
    locks can cause your serial code to be correct despite parallelism

What about performance?
  after all, we're probably locking as part of a plan to get parallel speedup

Locks and parallelism
  locks *prevent* parallel execution
  to get parallelism, you often need to split up data and locks
    in a way that lets each core use different data and different locks
    "fine grained locks"
  choosing best split of data/locks is a design challenge
    whole ph.c table; each table[] row; each entry
    whole FS; directory/file; disk block
    whole kernel; each subsystem; each object
  you may need to re-design code to make it work well in parallel
    example: break single free memory list into per-core free lists
      helps if threads were waiting a lot on lock for single free list
    such re-writes can require a lot of work!

Lock granularity advice
  start with big locks, e.g. one lock protecting entire module
    less deadlock since less opportunity to hold two locks
    less reasoning about invariants/atomicity required
  measure to see if there's a problem
    big locks are often enough -- maybe little time spent in that module
  re-design for fine-grained locking only if you have to

Let's look at locking in xv6.

A typical use of locks: ide.c
  typical of many O/S's device driver arrangements
  diagram:
    user processes, kernel, FS, iderw, append to disk queue
    IDE disk hardware
    ideintr
  sources of concurrency: processes, interrupt
  only one lock in ide.c: idelock -- fairly coarse-grained
  iderw() -- what does idelock protect?
    1. no races in idequeue operations
        2. if queue not empty, IDE h/w is executing head of queue
        3. no concurrent access to IDE registers
        ideintr() -- interrupt handler
        acquires lock -- might have to wait at interrupt level!
        uses idequeue (1)
        hands next queued request to IDE h/w (2)
        touches IDE h/w registers (3)

Atomic exchange instruction:
  mov $1, %eax
  xchg %eax, addr
  does this in hardware:
    lock addr globally (other cores cannot use it)
    temp = *addr
    *addr = %eax
    %eax = temp
    unlock addr
  x86 h/w provides a notion of locking a memory location
    different CPUs have had different implementations
    diagram: cores, bus, RAM, lock thing
    so we are really pushing the problem down to the hardware
    h/w implements at granularity of cache-line or entire bus
  memory lock forces concurrent xchg's to run one at a time, not interleaved

Now:
  acquire(l){
    while(1){
      if(xchg(&l->locked, 1) == 0){
        break
      }
    }
  }
  if l->locked was already 1, xchg sets to 1 (again), returns 1,
    and the loop continues to spin
  if l->locked was 0, at most one xchg will see the 0; it will set
    it to 1 and return 0; other xchgs will return 1
  this is a "spin lock", since waiting cores "spin" in acquire loop

Look at xv6 spinlock implementation
  spinlock.h -- you can see "locked" member of struct lock
  spinlock.c / acquire():
    see while-loop and xchg() call
    what is the pushcli() about?
      why disable interrupts?
  release():
    sets lk->locked = 0
    and re-enables interrupts

Detail: memory read/write ordering
  suppose two cores use a lock to guard a counter, x
  and we have a naive lock implementation
  Core A:          Core B:
    locked = 1
    x = x + 1      while(locked == 1)
    locked = 0       ...
                   locked = 1
                   x = x + 1
                   locked = 0
  the compiler AND the CPU re-order memory accesses
    i.e. they do not obey the source program's order of memory references
    e.g. the compiler might generate this code for core A:
      locked = 1
      locked = 0
      x = x + 1
      i.e. move the increment outside the critical section!
    the legal behaviors are called the "memory model"
  release()'s call to __sync_synchronize() prevents re-order
    compiler won't move a memory reference past a __sync_synchronize()
    and (may) issue "memory barrier" instruction to tell the CPU
  acquire()'s call to xchg() has a similar effect:
    intel promises not to re-order past xchg instruction
    some junk in x86.h xchg() tells C compiler not to delete or re-order
      (volatile asm says don't delete, "m" says no re-order)
  if you use locks, you don't need to understand the memory ordering rules
    you need them if you want to write exotic "lock-free" code

Why spin locks?
  don't they waste CPU while waiting?
  why not give up the CPU and switch to another process, let it run?
  what if holding thread needs to run; shouldn't waiting thread yield CPU?
  spin lock guidelines:
    hold spin locks for very short times
    don't yield CPU while holding a spin lock
  systems often provide "blocking" locks for longer critical sections
    waiting threads yield the CPU
    but overheads are typically higher
    you'll see some xv6 blocking schemes later

Advice:
  don't share if you don't have to
  start with a few coarse-grained locks
  instrument your code -- which locks are preventing parallelism?
  use fine-grained locks only as needed for parallel performance
  use an automated race detector