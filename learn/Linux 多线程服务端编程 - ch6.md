# muduo 网络库简介

[TOC]

## 安装

### 依赖

* cmake

​      版本不低于2.8

*  boost

### 全量安装

`cat /proc/version`

Linux version 2.6.32_1-16-0-0_virtio (root@yf-bce-36-64-36.yf01.baidu.com) (gcc version 4.4.6 20120305 (Red Hat 4.4.6-4) (GCC) ) #1 SMP Thu May 14 15:30:56 CST 2015

* 安装依赖的 cmake

​       a. 官网下载源码

​           3.9.0安装失败，直接安装2.8了

* 安装 boost 库

修改project-config.jam，指定 gcc 位置

./b2 --prefix=/home/users/caiguanyang/tools/boost --build-type=minimal install 

* 执行 build.sh

修改脚本，先不执行 make

* 修改生成的build/release 目录中的 CMakeCache.txt中的 C 和C++编译器路径，指定为高版本的GCC，和 boost 库的位置
* ​



错误：

```c
Scanning dependencies of target muduo_base
[  1%] Building CXX object muduo/base/CMakeFiles/muduo_base.dir/AsyncLogging.cc.o
/home/users/caiguanyang/learn/muduo/muduo-master/muduo/base/AsyncLogging.cc:1: error: bad value (native) for -march= switch
/home/users/caiguanyang/learn/muduo/muduo-master/muduo/base/AsyncLogging.cc:1: error: bad value (native) for -mtune= switch
make[2]: *** [muduo/base/CMakeFiles/muduo_base.dir/AsyncLogging.cc.o] Error 1
make[1]: *** [muduo/base/CMakeFiles/muduo_base.dir/all] Error 2
make: *** [all] Error 2
```

### 仅安装 base 和 net 库

*考虑仅编译 base 和 net 库，其他的不编译*

http://blog.csdn.net/ljq550000/article/details/40869073

**前提：安装了 cmake 和 boost 库**

新建了个工程目录，拷贝 base 和 net 目录中用到的头文件和源码，自定义 CMakeLists.txt 文件

目录结构如下：

-muduo_src

-CMakeLists.txt

-muduo

--base

---CMakeListx.txt

---Thread.h

---Thread.cc

---...

--net

---CMakeLists.txt

---EventLoop.h

---…

在 muduo_src目录下执行 cmake .  && make 命令

**muduo_src 目录下的 CMakeLists.txt**

```cmake
cmake_minimum_required (VERSION 2.8)
# 设置 GCC, G++版本
set (CMAKE_C_COMPILER "/opt/compiler/gcc-4.8.2/bin/gcc")
set (CMAKE_CXX_COMPILER "/opt/compiler/gcc-4.8.2/bin/g++")

# 项目信息
project (muduo_cai)

# 设置头文件检索路径
include_directories ("${PROJECT_SOURCE_DIR}")
include_directories ("/home/users/caiguanyang/tools/boost/include")

# 生成 muduo 原生库
add_subdirectory (muduo/base)
add_subdirectory (muduo/net)
```

**muduo/base 目录下的 CMakeLists.txt**

```cmake
set (base_SRCS
  AsyncLogging.cc
  Condition.cc
  CountDownLatch.cc
  Date.cc
  Exception.cc
  FileUtil.cc
  LogFile.cc
  Logging.cc
  LogStream.cc
  ProcessInfo.cc
  Timestamp.cc
  TimeZone.cc
  Thread.cc
  ThreadPool.cc
)

# 生成链接库
add_library(muduo_base ${base_SRCS})
target_link_libraries(muduo_base pthread rt)
```

**muduo/net 目录下的 CMakeLists.txt**

```cmake
include(CheckFunctionExists)

check_function_exists(accept4 HAVE_ACCEPT4)
if(NOT HAVE_ACCEPT4)
  set_source_files_properties(SocketsOps.cc PROPERTIES COMPILE_FLAGS "-DNO_ACCEPT4")
endif()

set(net_SRCS
  Acceptor.cc
  Buffer.cc
  Channel.cc
  Connector.cc
  EventLoop.cc
  EventLoopThread.cc
  EventLoopThreadPool.cc
  InetAddress.cc
  Poller.cc
  poller/DefaultPoller.cc
  poller/EPollPoller.cc
  poller/PollPoller.cc
  Socket.cc
  SocketsOps.cc
  TcpClient.cc
  TcpConnection.cc
  TcpServer.cc
  Timer.cc
  TimerQueue.cc
  )

add_library(muduo_net ${net_SRCS})
target_link_libraries(muduo_net muduo_base)
```

