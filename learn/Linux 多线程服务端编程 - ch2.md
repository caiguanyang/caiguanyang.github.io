# Linux 多线程服务端编程

[TOC]

## 第二章：线程同步精要

### 资料

《Real-World Concurrency》

### 知识点

**CPU Cache**

http://cenalulu.github.io/linux/all-about-cpu-cache/

**锁引发的性能衰退**

http://blog.csdn.net/panaimin/article/details/5981766

**条件变量的虚假唤醒**

https://en.wikipedia.org/wiki/Spurious_wakeup

https://www.douban.com/note/309639829/





### 内容备注

1）只使用非递归的 mutex(非可重入)，否则需要重新考虑下你的设计。虽然可重入锁不会出现一个线程被自己锁死的情况，但是会存在其他隐藏的问题

2）使用 mutex 时，尽量通过 scoped Locking, 缩小临界区的范围和锁持有时间，并且采用 MutexGard 来处理锁定和解锁，避免忘记释放锁

3）【死锁】如果两个调用序列加锁的顺序不一致，则很可能导致死锁。

4）互斥锁是加锁原语，用来排他性地访问共享数据，它不是等待原语。如果需要等待一个条件成立，我们应该使用条件变量。





