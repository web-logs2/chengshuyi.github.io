---
title: "Notes8 中断、系统调用和异常"
date: 2020-04-25T13:36:12+08:00
description: ""
draft: true
tags: []
categories: []
---



why does hw want attention now?
  MMU cannot translate address
  User program divides by zero
  User program wants to execute kernel code (INT)
  Network hardware wants to deliver a packet
  Timer hardware wants to deliver a "tick"
  kernel CPU-to-CPU communication, e.g. to flush TLB (IPI)

these "traps" fall broadly speaking in 3 classes:
  Exceptions (page fault, divide by zero)
  System calls (INT, **intended exception**)
  Interrupts (devices want attention)
  (Warning: terminology isn't used consistently)

where do device interrupts come from?
  diagram:
    CPUs, LAPICs, IOAPIC, devices
    data bus
    interrupt bus
  the interrupt tells the kernel the device hardware wants attention
  the driver (in the kernel) knows how to tell the device to do things
  often the interrupt handler calls the relevant driver
    but other arrangements are possible (schedule a thread; poll)

how does trap() know which device interrupted?
  i.e. where did tf->trapno == T_IRQ0 + IRQ_TIMER come from?
  kernel tells LAPIC/IOAPIC what vector number to use, e.g. timer is vector 32
    page faults &c also have vectors (as we saw in last lecture)
    LAPIC / IOAPIC are standard pieces of PC hardware
    one LAPIC per CPU
  **IDT associates an instruction address with each vector number**
    IDT format is defined by Intel, configured by kernel
  each vector jumps to alltraps
  CPU sends many kinds of traps through IDT
    low 32 IDT entries have special fixed meaning
  xv6 sets up system calls (IRQ) to use IDT entry 64 (0x40)
  the point: the vector number reveals the source of the interrupt

diagram:
  IRQ or trap, IDT table, vectors, alltraps
  IDT:
    0: divide by zero
    13: general protection
    14: page fault
    32-255: device IRQs
    32: timer
    33: keyboard
    46: IDE
    64: INT

**how does the hardware know what stack to use for an interrupt?**
  when it switches from user space to the kernel
  hardware-defined TSS (task state segment) lets kernel configure CPU 
    one per CPU
    so each CPU can run a different process, take traps on different stacks

Let's talk about homework, which involves:
  interrupts + system calls
  challenges:
    get it to work at all
    maintain isolation (unfortunately, not easy way to test!)

**device interrupts arrive just like INT and pagefault**
  h/w pushes esp and eip on kernel stack
  s/w saves other registers, into a trapframe
  vector, alltraps, trap()

Polling versus interrupts
  Polling rather than interrupting, for high-rate devices
  Interrupt for low-rate devices, e.g. keyboard
    constant polling would waste CPU
  **Switch between polling and interrupting automatically**
    interrupt when rate is low (and polling would waste CPU cycles)
    poll when rate is high  (and interrupting would waste CPU cycles)
  Faster forwarding of interrupts to user space
    for page faults and user-handled devices
    h/w delivers directly to user, w/o kernel intervention?
    faster forwarding path through kernel?
  We will be seeing many of these topics later in the course