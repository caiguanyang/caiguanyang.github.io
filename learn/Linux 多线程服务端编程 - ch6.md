# muduo 网络库简介

[TOC]

## 安装

### 依赖

* cmake

​      版本不低于2.8

* ​

### mac 环境安装

* 安装依赖 cmake

​     a. 官网下周 cmake源码(适用与 mac 的 dgm 包)

https://cmake.org/download/

​      b. 安装

​      c. 执行如下命令`sudo "/Applications/CMake.app/Contents/bin/cmake-gui" --install`，可以在命令行使用

* 安装 boost 库

http://www.boost.org/users/download/

* Cmake 文件修改

### Linux环境安装

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



*考虑仅编译 base 和 net 库，其他的不编译*

http://blog.csdn.net/ljq550000/article/details/40869073





