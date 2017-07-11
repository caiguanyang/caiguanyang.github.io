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

**1.3 读写锁、互斥量性能对比**

[http://blog.chinaunix.net/uid-28852942-id-3756043.html](http://blog.chinaunix.net/uid-28852942-id-3756043.html)

互斥量毕竟只有两种状态，性能还是比较高的

​       在写操作比较多，或者本身同步的地方不多，建议使用互斥量

​       读操作远多于写操作的地方，建议用读写锁

​       如果临界区很小，执行的时间很短，可以考虑采用自旋锁

### 内容备注

1）对象构造要做到线程安全，唯一要求是在构造期间不要泄露this 指针

​     构造期间对象还没有完全初始化，是个半成品，暴露出去被其他实例使用，可能导致未知结果

2）作为 class 成员的 mutexlock 只能用于同步本 class 的其他数据成员的读和写，不能保护完全地析构（析构过程和函数调用时获取 mutexlock 存在 race condition，析构如果先执行的化，最终 mutexlock 会被销毁，函数调用可能一直阻塞，或者 core dump）

3）一个动态创建的对象是否还活着，光看指针是看不出来的，指针只是指向了一块内存，内存上的对象可能已经被销毁