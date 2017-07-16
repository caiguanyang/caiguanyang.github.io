# 多线程服务器的适用场合与常用编程模型

[TOC]

## 编程模型

### 单线程编程模型

#### Reactor模式

non-blocking IO, IO multiplexing

参考：http://www.blogjava.net/DLevin/archive/2015/09/02/427045.html

事件驱动，有一个或多个并发输入源，有一个 ServiceHandler, 多个 Request Handlers

**缺点：**共享一个 Reactor 的多个 channel，如果出现一个channel 的 IO读写耗时过长，则会影响其他 channel 的耗时，如处理大文件传输。



### 多线程编程模型

#### Thread Per Connection

**缺点：**线程的切换、同步、数据转移会引发性能问题；数据的同步也会导致编程比较复杂；可移植性不好。

当然如果对于性能要求不高，且各线程处理的逻辑比较简单，不涉及数据共享，还是可以考虑的。且如果 IO耗时比较长，它也是个不错的选择。



### 知识点

#### non-blocking io

http://www.kegel.com/dkftpbench/nonblocking.html

poll

