




## 安装编译环境

采用alinux3操作系统，应该也适用于redhat/centos系列：

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


## 编译libbpf & bcc

```shell
cd bpftrace
git submodule update --init
mkdir build
cd build
../build-libs.sh
```

## 编译bpftrace

### 构建release版本
`cmake -DCMAKE_BUILD_TYPE=Release -DBUILD_TESTING=OFF .. `

### 构建debug版本呢
`cmake -DCMAKE_BUILD_TYPE=Debug -DBUILD_TESTING=OFF ..`



## 代码提交

需要运行 git clang-format，注意：这个命令需要未提交代码，不然会出现 no modified files to format错误