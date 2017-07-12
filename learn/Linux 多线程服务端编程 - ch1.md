# Linux 多线程服务端编程



## 第一章：线程安全的对象生命周期管理

[TOC]



### 知识点

**1.1 线程间共享的内容：**

[http://blog.csdn.net/xiaominkong123/article/details/52099048](http://blog.csdn.net/xiaominkong123/article/details/52099048)

​     线程有自己的栈信息，存储局部变量，函数调用等信息（还有错误码）

**1.2 智能指针**

shared_ptr

weak_ptr

[http://www.boost.org/doc/libs/1_64_0/libs/smart_ptr/shared_ptr.htm](http://www.boost.org/doc/libs/1_64_0/libs/smart_ptr/shared_ptr.htm)

  关注 shared_ptr 的线程安全特征：不同线程可以同时修改不同实例（即使关联的是一个对象）

如何将 this 对象转成 shared_ptr 呢？

   参考 boost::enable_shared_from_this



**1.3 读写锁、互斥量性能对比**

[http://blog.chinaunix.net/uid-28852942-id-3756043.html](http://blog.chinaunix.net/uid-28852942-id-3756043.html)

互斥量毕竟只有两种状态，性能还是比较高的

​       在写操作比较多，或者本身同步的地方不多，建议使用互斥量

​       读操作远多于写操作的地方，建议用读写锁

​       如果临界区很小，执行的时间很短，可以考虑采用自旋锁



**1.4 boost::bind**



### 内容备注

1）对象构造要做到线程安全，唯一要求是在构造期间不要泄露this 指针

​     构造期间对象还没有完全初始化，是个半成品，暴露出去被其他实例使用，可能导致未知结果

2）作为 class 成员的 mutexlock 只能用于同步本 class 的其他数据成员的读和写，不能保护完全地析构（析构过程和函数调用时获取 mutexlock 存在 race condition，析构如果先执行的化，最终 mutexlock 会被销毁，函数调用可能一直阻塞，或者 core dump）

3）一个动态创建的对象是否还活着，光看指针是看不出来的，指针只是指向了一块内存，内存上的对象可能已经被销毁

4）C++中出现的内存问题：缓冲区溢出，空悬指针/野指针，内存泄露，重复释放，不配对的 new/delete，内存碎片；采用 std::vector, 智能指针可以解决前5个问题

5）通常 factory 对象是个 singleton，在程序正常运行期间不会被销毁

6）尽管本章通篇都在讲如何安全地使用（包括析构）跨线程的对象，但我建议尽量减少使用跨线程的对象：用流水线，生产者消费者，任务队列这些有规律的机制，最低限度地共享数据。这是最好的多线程编程建议。

7）【弱回调】如果对象还活着，就调用它的成员函数，否则忽略之；可以采用 weak_ptr 实现。

​        参考 recipes/thread/WeakCallback.h



### 示例说明

从观察者模式中 observers 的调用，到对象池，主要讲解了智能指针的使用，建议大家采用智能指针来减少内存问题，做到线程安全的对象回调与析构





