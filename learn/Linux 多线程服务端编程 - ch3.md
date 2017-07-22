# 多线程服务器的适用场合与常用编程模型

[TOC]

## 介绍

“多线程”的价值，我认为是为了更好地发挥多核处理器的效能；

blocking-queue 是多线程编程的利器，可用于实现任务队列，生产者消费者队列



## 编程模型

### 单线程编程模型event loop

#### Reactor模式

参考：http://www.blogjava.net/DLevin/archive/2015/09/02/427045.html

事件驱动，有一个或多个并发输入源，有一个 ServiceHandler, 多个 Request Handlers

<img src="https://caiguanyang.github.io/img/reactor_1.png" width="500" height="250" align=center alt="reactor 类图" />



**缺点:**共享一个 Reactor 的多个 channel，如果出现一个channel 的 IO读写耗时过长，则会影响其他 channel 的耗时，如处理大文件传输。另外基于事件驱动的编程模型，要求事件回调必须是非阻塞的，对于涉及网络 IO的请求响应式协议，它容易割裂业务逻辑，使其散步与多个回调函数中，不容易理解和维护

   有关 event loop，比较明显的缺点是**非抢占式的**

 

#### 采用此模式的应用

1）lighttpd

2）Twisted



### 多线程编程模型

#### Thread Per Connection

**缺点：**线程的切换、同步、数据转移会引发性能问题；数据的同步也会导致编程比较复杂；可移植性不好。

当然如果对于性能要求不高，且各线程处理的逻辑比较简单，不涉及数据共享，还是可以考虑的。且如果 IO耗时比较长，它也是个不错的选择。

它不适合高并发的场合，其 scalability 不佳。

#### one loop per thread

每个 IO线程有一个 event loop(或者 Reactor)，用于处理读写和定时事件。

#### 线程池

用于光有计算任务的线程

#### 适用多线程程序的场景

有多个 CPU 可用；

提供非均质的服务（事件的响应有优先级差异）；

利用异步操作，如 logging;

提高响应速度，让 IO 和“计算”相互交叠，降低 latency（把 IO 操作通过 BlockingQueue交给其他线程处理，自己不必等待）。虽然多线程不能提高绝对性能，但能提高平均响应性能。

多线程能有效划分责任与功能，让每个线程的逻辑比较简单，任务单一，便于编码。



### 知识点

#### non-blocking io

http://www.kegel.com/dkftpbench/nonblocking.html

#### select、poll、epoll

http://www.jianshu.com/p/dfd940e7fca2

可以从支持一个进程所能打开的最大连接数，fd增加后所带来的效率问题，消息传递方式区分3种方式

应用：http://xuding.blog.51cto.com/4890434/1739649

#### linux的5种 IO 模型

http://www.jianshu.com/p/486b0965c296

#### 网络编程中的基本概念

http://www.jianshu.com/p/486b0965c296

https://www.zybuluo.com/phper/note/595507



#### blocking queue







## 进程间通信

### 进程间通信首选 socket

​        TCP 协议的一个天生好处是“可记录，可重现”，可用 tcpdump, wireshark 解决两个进程间协议和状态争端的好帮手，也是性能分析的利器;

​        支持跨语言; 

​        支持跨机器通信；

​        分布式系统中采用 TCP 长连接有两个好处：1）一个容易定位分布式系统中服务之间的依赖关系  2）通过接收和发送队列的长度也较容易定位网络或者程序故障（netstat 打印的 Recv-Q 和 Send-Q）。

### 共享内存



### Unix domain socket协议

nginx和php-fpm之间的通信

参考：http://www.cleey.com/blog/single/id/856.html

虽然 socket 也能用于同机通信，但是 unix domain socket 更有效率：不需要经过网络协议栈；不需要打包拆包，计算校验和等；只是将数据从一个进程拷贝的另一个进程

