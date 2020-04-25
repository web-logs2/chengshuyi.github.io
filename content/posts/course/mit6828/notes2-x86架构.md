---
title: "Notes2 X86架构"
date: 2020-04-25T13:35:11+08:00
description: ""
draft: true
tags: [计算机课程笔记,操作系统]
categories: [计算机课程笔记,操作系统]
---

### PC architecture

- A full PC has:

  - an x86 CPU with registers, execution unit, and memory management
  - CPU chip pins include address and data signals
  - memory
  - disk
  - keyboard
  - display
  - other resources: BIOS ROM, clock, ...

- Instructions are in memory too!

  - IP - instruction pointer (PC on PDP-11, everything else)
  - increment after running each instruction
  - **can be modified by CALL, RET, JMP, conditional jumps**

- Still not interesting - need I/O to interact with outside world

  - Original PC architecture: use dedicated

    I/O space

    - Works same as memory accesses but set I/O signal

    - Only 1024 I/O addresses

    - Accessed with special instructions (IN, OUT)

    - Example: write a byte to line printer:

      ```c
      #define DATA_PORT    0x378
      #define STATUS_PORT  0x379
      #define   BUSY 0x80
      #define CONTROL_PORT 0x37A
      #define   STROBE 0x01
      void
      lpt_putc(int c)
      {
        /* wait for printer to consume previous byte */
        while((inb(STATUS_PORT) & BUSY) == 0)
          ;
      
        /* put the byte on the parallel lines */
        outb(DATA_PORT, c);
      
        /* tell the printer to look at the data */
        outb(CONTROL_PORT, STROBE);c
        outb(CONTROL_PORT, 0);
      }	
      ```

  - **Memory-Mapped I/O**

    - Use normal physical memory addresses
      - Gets around limited size of I/O address space
      - No need for special instructions
      - System controller routes to appropriate device
    - Works like ``magic'' memory:
      - *Addressed* and *accessed* like memory, but ...
      - ... does not *behave* like memory!
      - Reads and writes can have ``side effects''
      - Read results can change due to external events

- What if we want to use more than 2^16 bytes of memory?

  - 8086 has 20-bit physical addresses, can have 1 Meg RAM
  - **the extra four bits usually come from a 16-bit "segment register":**
  - CS - code segment, for fetches via IP
  - SS - stack segment, for load/store via SP and BP
  - DS - data segment, for load/store via other registers
  - ES - another data segment, destination for string operations
  - virtual to physical translation: pa = va + seg*16
  - e.g. set CS = 4096 to execute starting at 65536
  - tricky: can't use the 16-bit address of a stack variable as a pointer
  - a *far pointer* includes full segment:offset (16 + 16 bits)
  - tricky: pointer arithmetic and array indexing across segment boundaries

### x86 Physical Memory Map

参考实验1的笔记

### x86 Instruction Set

参考实验1的笔记

### gcc x86 calling conventions

- GCC dictates how the stack is used. Contract between caller and callee on x86:

  - at entry to a function (i.e. just after call):
    - %eip points at first instruction of function
    - %esp+4 points at first argument
    - %esp points at return address
  - after ret instruction:
    - %eip contains return address
    - %esp points at arguments pushed by caller
    - called function may have trashed arguments
    - %eax (and %edx, if return type is 64-bit) contains return value (or trash if function is `void`)
    - %eax, %edx (above), and %ecx may be trashed
    - %ebp, %ebx, %esi, %edi must contain contents from time of `call`
  - Terminology:
    - %eax, %ecx, %edx are "caller save" registers
    - %ebp, %ebx, %esi, %edi are "callee save" registers

- Functions can do anything that doesn't violate contract. By convention, GCC does more:

  - each function has a stack frame marked by %ebp, %esp

    ```
    		       +------------+   |
    		       | arg 2      |   \
    		       +------------+    >- previous function's stack frame
    		       | arg 1      |   /
    		       +------------+   |
    		       | ret %eip   |   /
    		       +============+   
    		       | saved %ebp |   \
    		%ebp-> +------------+   |
    		       |            |   |
    		       |   local    |   \
    		       | variables, |    >- current function's stack frame
    		       |    etc.    |   /
    		       |            |   |
    		       |            |   |
    		%esp-> +------------+   /
    		
    ```

  - %esp can move to make stack frame bigger, smaller

  - %ebp points at saved %ebp from previous function, chain to walk stack

  - function prologue:

    ```
    			pushl %ebp
    			movl %esp, %ebp
    ```

    or

    ```
    			enter $0, $0
    ```

    enter usually not used: 4 bytes vs 3 for pushl+movl, not on hardware fast-path anymore

  - function epilogue can easily find return EIP on stack:

    ```
    			movl %ebp, %esp
    			popl %ebp
    ```

    or

    ```
    			leave
    ```

    leave used often because it's 1 byte, vs 3 for movl+popl

- Big example:

  - C code

    ```
    		int main(void) { return f(8)+1; }
    		int f(int x) { return g(x); }
    		int g(int x) { return x+3; }
    		
    ```

  - assembler

    ```assembly
    		_main:
    					prologue
    			pushl %ebp
    			movl %esp, %ebp
    					body
    			pushl $8
    			call _f
    			addl $1, %eax
    					epilogue
    			movl %ebp, %esp
    			popl %ebp
    			ret
    		_f:
    					prologue
    			pushl %ebp
    			movl %esp, %ebp
    					body
    			pushl 8(%esp)
    			call _g
    					epilogue
    			movl %ebp, %esp
    			popl %ebp
    			ret
    
    		_g:
    					prologue
    			pushl %ebp
    			movl %esp, %ebp
    					save %ebx
    			pushl %ebx
    					body
    			movl 8(%ebp), %ebx
    			addl $3, %ebx
    			movl %ebx, %eax
    					restore %ebx
    			popl %ebx
    					epilogue
    			movl %ebp, %esp
    			popl %ebp
    			ret
    ```

- Super-small`_g`:

  ```assembly
  		_g:
  			movl 4(%esp), %eax
  			addl $3, %eax
  			ret
  ```

- Shortest `_f`?

- Compiling, linking, loading:

  - *Preprocessor* takes C source code (ASCII text), expands #include etc, produces C source code
  - *Compiler* takes C source code (ASCII text), produces assembly language (also ASCII text)
  - *Assembler* takes assembly language (ASCII text), produces `.o` file (binary, machine-readable!)
  - *Linker* takes multiple '`.o`'s, produces a single *program image* (binary)
  - *Loader* loads the program image into memory at run-time and starts it executing