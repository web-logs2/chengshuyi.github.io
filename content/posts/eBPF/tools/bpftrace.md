---
title: "eBPF 开源项目之 bpftrace"
date: 2023-04-22T14:08:58+08:00 
description: ""
draft: true
tags: [eBPF, bpftrace]
categories: [eBPF]
---


## 简介

bpftrace 是一个高级的动态跟踪工具，它基于 Linux 内核的 eBPF 技术，可以在不修改内核代码的情况下，对内核进行高效、安全、可扩展的跟踪和分析。bpftrace 通过提供类 C 语言的脚本语言和一系列内置的函数，使得用户可以方便地实现各种跟踪和诊断需求，例如统计系统调用次数、查找瓶颈、检测安全漏洞、诊断性能问题等。bpftrace 是 Linux 系统管理员和开发人员的有力工具，可以提高效率、缩短故障排查时间、优化系统性能。

## 安装 bpftrace

bpftrace 可以通过官方提供的安装包安装，也可以自行源码编译安装。

### 安装包安装

bpftrace 官方提供了 ubuntu、Fedora、Gentoo、Debian、openSUSE 等内核发行版的 bpftrace 安装包，下面是常见发行版的 bpftrace 安装方法：

- ubuntu：`sudo apt-get install -y bpftrace`，需要注意的是 ubuntu 的内核版本最低要求为 19.04
- Fedora：`sudo dnf install -y bpftrace`，需要注意的是 Fedora 最低版本要求是 28
- Gentoo：`sudo emerge -av bpftrace`
- Debian：安装包可以从该链接下载：https://tracker.debian.org/pkg/bpftrace
- openSUSE：安装包可以从该链接下载：https://software.opensuse.org/package/bpftrace
- CentOS：centos 安装包由个人用户维护，连接是：https://github.com/fbs/el7-bpf-specs/blob/master/README.md#repository
- Arch：`sudo pacman -S bpftrace`，可以很轻松地从官方仓库下载
- Alpine：`sudo apk add bpftrace`，`sudo apk add bpftrace-doc bpftrace-tools bpftrace-tools-doc` 安装 bpftrace 的工具和文档

### 源码编译安装

本文采用 alinux3 操作系统，应该也适用于 redhat/centos 系列。首先安装依赖库：
```shell
yum install -y flex bison cmake make git gcc-c++
yum install -y elfutils-libelf-devel elfutils-libs
yum install -y lrzsz binutils-devel
yum install -y gtest-devel gmock-devel libpcap-devel libpcap
yum install -y cereal-devel
yum install -y elfutils-devel llvm llvm-devel
yum install -y clang clang-devel
yum install -y iperf3 luajit luajit-devel netperf
yum install -y elfutils-debuginfod-0.180-1.1.al8.x86_64
yum install -y bcc-devel
```

然后，下载依赖库 libbpf 和 bcc 源码并进行编译：

```shell
cd bpftrace
git submodule update --init
mkdir build
cd build
../build-libs.sh
```

最后编译 bpftrace，我们可以通过 cmake 的 CMAKE_BUILD_TYPE 变量来控制编译 release 版本还是 debug 版本。编译 release 版本：`cmake -DCMAKE_BUILD_TYPE=Release -DBUILD_TESTING=OFF ..` 构建 debug 版本：`cmake -DCMAKE_BUILD_TYPE=Debug -DBUILD_TESTING=OFF ..`

`make -j32 & make install`



## bpftrace 常用命令


### bpftrace --info 查看系统支持情况

bpftrace --info 命令可以查看

- System：系统信息
- Build：bpftrace 构建信息，比如当前依赖的 llvm 版本
- Kernel Hepers：是否支持某些 eBPF 辅助函数，可以看到我们的系统支持 `probe_read` 等很多辅助函数
- Kernel features：是否支持某些 eBPF 特性，如 BTF、指令数限制等
- Map types：支持的 eBPF map 类型
- Probe types：支持的探测类型

```shell
System
  OS: Linux 5.10.84-10.4.al8.x86_64 #1 SMP Tue Apr 12 12:31:07 CST 2022
  Arch: x86_64

Build
  version: v0.16.0-179-g5c34-dirty
  LLVM: 13.0.1
  unsafe uprobe: no
  bfd: yes
  libdw (DWARF support): yes

Kernel helpers
  probe_read: yes
  probe_read_str: yes
  probe_read_user: yes
  probe_read_user_str: yes
  probe_read_kernel: yes
  probe_read_kernel_str: yes
  get_current_cgroup_id: yes
  send_signal: yes
  override_return: no
  get_boot_ns: yes
  dpath: yes
  skboutput: yes

Kernel features
  Instruction limit: 1000000
  Loop support: yes
  btf: yes
  module btf: no
  map batch: yes
  uprobe refcount (depends on Build:bcc bpf_attach_uprobe refcount): yes

Map types
  hash: yes
  percpu hash: yes
  array: yes
  percpu array: yes
  stack_trace: yes
  perf_event_array: yes

Probe types
  kprobe: yes
  tracepoint: yes
  perf_event: yes
  kfunc: yes
  iter:task: yes
  iter:task_file: yes
  iter:task_vma: no
  kprobe_multi: no
  raw_tp_special: yes
```


### bpftrace -l 列出探针

我们可以通过 bpftrace -l 列出 bpftrace 当前支持的所有探针。也可以通过通配符的方式来查看某类探针，比如 `bpftrace -l 'tracepoint:syscalls:sys_enter_*'` 会列出前缀是 sys_enter_的所有探测点。

### bpftrace -e 执行单行命令

`bpftrace -e 'program'` 可以通过简单的一行命令，来执行我们的程序。比如下面我们通过一行命令来获取进程的系统调用计数。

```shell
bpftrace -e 'tracepoint:raw_syscalls:sys_enter { @[comm] = count (); }'
Attaching 1 probe...
^C

@[gmain]: 2
@[tuned]: 2
@[in:imjournal]: 6
@[nscd]: 12
@[docker]: 22

```

### bpftrace [FILENAME] 执行脚本

bpftrace 后面带着脚本名字，bpftrace 会立即执行该脚本。

biolatency.bt

```shell
bpftrace biolatency.bt
Attaching 5 probes...
Tracing block device I/O... Hit Ctrl-C to end.
^C


@usecs: 
[256, 512)             1 |@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@|
[512, 1K)              1 |@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@|
```

## bpftrace one-liners

bpftrace one-liners 是指一种简单的 bpftrace 脚本，可以通过单行命令实现对系统性能和行为的实时监测。通过 `-e` 参数来指定执行的脚本程序。下面是一个 bpftrace 单行命令，可以实现监测哪个进程打开了哪个文件并进行打印输出：

```shell
bpftrace -e 'tracepoint:syscalls:sys_enter_openat { printf ("% s % s\n", comm, str (args->filename)); }'
Attaching 1 probe...
sed /lib64/libpthread.so.0
sed /proc/filesystems
```

## bpftrace 常用语法


bpftrace 是一种用于动态跟踪 Linux 内核的工具，它使用一种类似于 awk 的语法来编写跟踪脚本。以下是 bpftrace 的语法简介：

### 注释

bpftrace 的注释使用 `//` 符号开头的行表示单行注释，bpftrace 会忽略这些行。如果需要添加多行注释，则可以使用 `/* */` 进行多行注释。这些注释的作用是为了帮助代码的可读性和可维护性，同时也方便开发者添加对代码的解释和说明。

### 语法结构

bpftrace 跟踪脚本的基本语法结构如下：

```
ProbeType:AttachPoint
{
    //statements
}
```

其中，ProbeType 表示探测类型，AttachPoint 表示探测点，它们之间通过冒号隔开。因为一个探测类型会对应多个探测点，所以需要通过 AttachPoint 来指定具体的探测点。例如，`kprobe:tcp_sendmsg` 表示探测类型是 kprobe，探测点是 `tcp_sendmsg`，即跟踪内核中 `tcp_sendmsg` 函数的执行情况。statements 是指需要执行的操作，可以是打印输出、计数器统计、变量赋值等操作。多个 statements 之间需要用分号隔开。通过使用这些语句，我们可以在 BPF 程序中监控和分析系统的行为、性能等信息，从而进行系统性能调优、故障排除等工作。


### 过滤器

bpftrace 过滤器的主要功能是对探测中得到的数据进行筛选和过滤，只选择符合特定条件的数据进行处理和分析。过滤器通常用于在大量数据中筛选出特定的数据进行分析和处理，从而提高程序的性能和效率。bpftrace 过滤器支持多种数据类型的比较和筛选，如整型、浮点型、字符串等。可以使用各种比较运算符（如 ==、>、<、>=、<=、!= 等）和逻辑运算符（如 &&、||、! 等）进行筛选。同时，bpftrace 过滤器还支持使用正则表达式进行字符串匹配，更灵活地进行数据筛选。例如：
```c
// 跟踪某个进程的 sys_read 系统调用，只输出长度大于 10 的读取数据
tracepoint:syscalls:sys_enter_read /pid != 123 && args->count > 10 /
{
  printf ("read data (count=% d): \n", args->count);
}
```

### 变量和表达式

bpftrace 的变量和表达式是用于存储和计算数据的重要元素。变量是用于存储和处理数据的标识符，可以存储各种类型的数据，如整型、字符串等。在 bpftrace 中，变量名以 `$` 符号开头。这种语法风格来源于一些脚本语言，例如 bash shell 和 Perl 等，用于区分变量和常量量。

表达式是用于计算和操作数据的语句，可以使用各种算术运算符（如 +、-、*、/、% 等），比较运算符（如 ==、>、<、>=、<=、!= 等），逻辑运算符（如 &&、||、! 等）和位运算符（如 &、|、^ 等）进行计算和操作。表达式可以包括变量、常量和运算符，可以用于计算和处理各种类型的数据。

bpftrace 的变量和表达式可以用于实现各种功能，如打印输出、计数器统计、变量赋值、条件判断和循环等等。通过使用变量和表达式，我们可以更灵活和高效地处理和分析数据，从而进行系统性能调优、故障排除等工作。


### 函数和内置变量 

pftrace 提供了大量的函数和内置变量，方便用户进行数据分析和处理。函数和内置变量可以在 bpftrace 程序中直接使用，无需用户自己定义和实现，可以节省用户的时间和精力，同时也提高了程序的可读性和可维护性。具体可见：https://github.com/iovisor/bpftrace/blob/master/docs/reference_guide.md#functions

### eBPF map 相关函数

bpftrace map 函数是一种内置的操作函数，用于在 BPF 程序中创建和操作哈希表（map）。哈希表是一种由键值对（key-value pairs）组成的数据结构，可以快速地查找和存储数据。bpftrace map 函数提供了一系列的操作，如计数、求和、平均值、最大和最小值、直方图等，可以用于在 eBPF 程序中处理和分析数据。bpftrace map 函数在 eBPF 编程中非常常用，可以帮助开发者更方便地操作和处理数据。它包括以下不同的操作：

- count ()：计算调用该函数的次数，比如：`bpftrace -e 'kprobe:vfs_read { @reads = count ();  }`，统计了内核函数 `vfs_read` 执行的次数。
- sum (int n)：求 n 的总和，比如：`bpftrace -e 'kprobe:vfs_read { @bytes [comm] = sum (arg2); }`，计算了每个进程的 IO 字节数。
- avg (int n)：求 n 的平均值，比如：`bpftrace -e 'kprobe:vfs_read { @bytes [comm] = avg (arg2); }`，计算了每个进程平均每次读取的字节数。
- min (int n)：记录 n 中的最小值。
- max (int n)：记录 n 中的最大值。
- stats (int n)：返回 n 的计数、平均值和总和。
- hist (int n)：生成 n 的对数直方图。
- lhist (int n, int min, int max, int step)：生成 n 的线性直方图。
- delete (@x [key])：删除作为参数传递的哈希表元素。
- print (@x [, top [, div]])：打印哈希表，可选择仅打印顶部条目和使用除数。
- print (value)：打印值。
- clear (@x)：从哈希表中删除所有键。
- zero (@x)：将哈希表中的所有值设置为零。

## bpftrace 官方工具

bpftrace 也提供了大量的 [通用工具](https://github.com/iovisor/bpftrace/tree/master/tools)，可以用于跟踪和分析 Linux 系统中的各种事件和行为。这些工具涵盖了很多方面，包括 I/O 监控、进程和系统监控、网络监控等等。每个工具都有其特定的功能和参数，用户可以根据需求选择合适的工具来进行跟踪和分析。根据不同的功能，可以将 bpftrace 提供的工具分成以下几类：

- I/O 监控工具：biolatency.bt，bitesize.bt，biosnoop.bt，dcsnoop.bt，mdflush.bt，opensnoop.bt，syncsnoop.bt，undump.bt，vfscount.bt，vfsstat.bt，writeback.bt，xfsdist.bt。
- 进程和系统监控工具：capable.bt，cpuwalk.bt，gethostlatency.bt，killsnoop.bt，loads.bt，oomkill.bt，pidpersec.bt，runqlat.bt，runqlen.bt，statsnoop.bt，syscount.bt，threadsnoop.bt。
- 网络监控工具：ssllatency.bt，sslsnoop.bt，tcpaccept.bt，tcpconnect.bt，tcplife.bt，tcpretrans.bt，tcpsynbl.bt，tcpdrop.bt。
- 其他工具：bashreadline.bt，biolatency-kp.bt，execsnoop.bt，naptime.bt，setuids.bt。

需要注意的是，这个分类并不是完全准确和全面的，因为很多工具可以同时涵盖多个功能。另外，bpftrace 提供的通用工具可以帮助用户快速展示系统的性能和行为，但并不是所有情况下都适用。因为每个应用场景都有其独特的需求和限制，通用的工具无法完全涵盖所有情况，这时候用户需要编写自己的跟踪脚本来满足特定的需求。

下面是对 bpftrace 提供的每个工具的简要描述：

| 工具名称            | 功能说明                                                     |
| ------------------- | ------------------------------------------------------------ |
| biolatency.bt        | 跟踪块设备的 I/O 操作延迟时间。                               |
| bitesize.bt          | 跟踪块设备的 I/O 操作大小。                                   |
| biosnoop.bt          | 跟踪块设备上的 I/O 操作，包括读写和同步异步操作。             |
| dcsnoop.bt           | 跟踪块设备上的延迟完整性数据（DC）操作，用于 SSD 和闪存设备。 |
| mdflush.bt           | 跟踪软件 RAID （md）设备上的 I/O 操作。                       |
| opensnoop.bt         | 跟踪文件打开操作，包括进程 ID、文件名等信息。                 |
| syncsnoop.bt         | 跟踪文件系统同步操作。                                       |
| undump.bt            | 跟踪内核中的解压缩操作。                                     |
| vfscount.bt          | 跟踪文件系统操作，统计并输出操作计数器。                     |
| vfsstat.bt           | 跟踪文件系统操作，输出文件系统性能统计信息。                 |
| writeback.bt         | 跟踪文件系统写回操作。                                       |
| xfsdist.bt           | 跟踪 XFS 文件系统的块分配和空间使用情况。                     |
| capable.bt           | 跟踪进程的权限检查操作。                                     |
| cpuwalk.bt           | 跟踪 CPU 上的进程调度操作。                                   |
| gethostlatency.bt    | 跟踪主机名解析操作的延迟时间。                               |
| killsnoop.bt         | 跟踪进程的 kill 操作。                                       |
| loads.bt             | 跟踪系统负载情况。                                           |
| oomkill.bt           | 跟踪系统 Out Of Memory （OOM）事件，输出导致 OOM 的进程信息。 |
| pidpersec.bt         | 跟踪进程创建速率。                                           |
| runqlat.bt           | 跟踪进程在运行队列中等待的时间。                             |
| runqlen.bt           | 跟踪进程运行队列的长度。                                     |
| statsnoop.bt         | 跟踪系统调用的状态信息。                                     |
| syscount.bt          | 跟踪系统调用的计数器。                                       |
| threadsnoop.bt       | 跟踪线程的创建和销毁操作。                                   |
| ssllatency.bt        | 跟踪 SSL 握手操作的延迟时间。                                |
| sslsnoop.bt          | 跟踪 SSL 握手操作的详细信息。                                |
| tcpaccept.bt         | 跟踪 TCP 连接的 accept 操作。                                |
| tcpconnect.bt        | 跟踪 TCP 连接的 connect 操作。                               |
| tcplife.bt           | 跟踪 TCP 连接的生命周期。                                    |
| tcpretrans.bt        | 跟踪 TCP 重传操作。                                           |
| tcpsynbl.bt          | 跟踪 TCP SYN 连接的状态。                                    |
| tcpdrop.bt           | 跟踪 TCP 连接的丢弃操作。                                    |
| bashreadline.bt      | 跟踪 Bash 的命令行输入操作。                                 |
| biolatency-kp.bt     | 跟踪块设备的 I/O 操作延迟时间，使用 kernel probe（kprobe）实现。 |
| execsnoop.bt         | 跟踪程序的执行操作。                                         |
| naptime.bt           | 跟踪进程的 sleep 操作。                                      |
| setuids.bt           | 跟踪进程的 setuid 和 setgid 操作。                            |

需要注意的是，这些工具的具体功能可能因版本更新、内核配置等因素而有所不同。建议在使用时参考相关的文档和手册，并根据实际情况进行调整和优化。