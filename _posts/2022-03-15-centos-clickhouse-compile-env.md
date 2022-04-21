---
layout: post
title: ClickHouse Centos编译与打包环境
date: 2022-03-15
Author: levy5307
tags: []
comments: true
toc: true
---

当前ClickHouse的官方文档中只有Ubuntu的编译环境搭建，没有Centos相关文档。这里根据个人的实际搭建经验，将Centos上搭建ClickHouse编译环境的步骤进行讲解（该教程在centos6和centos7.3上分别进行过实践验证）


## 编译

### 更新yum源

```
cd /etc/yum.repos.d/
sed -i 's/mirrorlist/#mirrorlist/g' /etc/yum.repos.d/CentOS-*
sed -i 's|#baseurl=http://mirror.centos.org|baseurl=http://vault.centos.org|g' /etc/yum.repo s.d/CentOS-*
yum update -y
```

### 安装wget

```
yum install wget -y
```

### 获取ldb-toolchain

```
mkdir env && cd env
wget https://github.com/amosbird/ldb_toolchain_gen/releases/download/v0.9.1/ldb_toolchain_gen.sh
sh ldb_toolchain_gen.sh /root/env/ldb_toolchain

export PATH=$PATH:/root/env/ldb_toolchain/bin
```

### 安装ccache

其实ccache不是必要的，只是会让后续的编译过程更快一些。

```
cd /root/env/ldb_toolchain/bin
wget https://github.com/levy5307/ldb_toolchain_gen/raw/main/ccache
chmod +x ccache
```

### 配置CC和CXX

```
export CC=clang CXX=clang++
```

### 安装perl

```
yum install perl -y
```

### 安装git

```
yum install git -y
```

### 配置使用Python3

```
cd /usr/bin
rm python
ln -s /root/env/ldb_toolchain/bin/ldb-python3 python
```

该步骤可能会导致yum无法使用，需要修改yum文件

```
vim /usr/bin/yum
```

```
#! /usr/bin/python
```
===>
```
#! /usr/bin/python2.6
```

### build源码

```
cd /root/ClickHouse
mkdir build && cd build
cmake .. -DENABLE_JEMALLOC=1
ninja
```

整个编译过程会比较久，大约需要2-3个小时，请耐心等待。

## 打包

ClickHouse内部没有支持CPack打包，需要对`CMakeLists.txt`和`git_status.cmake`文件做适当的修改。

`git_status.cmake`中添加:

```
execute_process(
    COMMAND ${GIT_EXECUTABLE} describe
    WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}
    OUTPUT_VARIABLE CLICKHOUSE_TAG
    OUTPUT_STRIP_TRAILING_WHITESPACE)
else()
```

`CMakeLists.txt`中添加: 

```
set(CPACK_PACKAGE_VERSION ${CLICKHOUSE_TAG})
set(CPACK_PACKAGE_VENDOR "Xiaomi")
install(PROGRAMS pack-bin/clickhouse_exporter DESTINATION ${CMAKE_INSTALL_BINDIR}) 	## 将clickhouse_exporter打包进去
install(PROGRAMS pack-bin/node_exporter DESTINATION ${CMAKE_INSTALL_BINDIR}) 		## 将node_exporter打包进去
include(CPack)
```

### 打包成TGZ包

```
cd build
cpack -G TGZ
```

### 打包成RPM包

```
yum install -y rpm-build
cd build
cpack -G RPM
```

## Dockerfile

如果你不想自己动手搭建环境，这里也有一个现有的[Dockerfile](https://github.com/levy5307/ClickHouse-Docker)可以使用，clone下来后执行：

```
docker build -t clickhouse-buildenv .
```

即可生成Docker镜像。

