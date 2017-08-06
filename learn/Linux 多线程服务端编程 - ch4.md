# C++多线程系统编程精要

[TOC]

Date: 2017-07-22

## 知识点

* 在没有适当同步的情况下，多个 CPU 上运行的多个线程中的事件发生先后顺序是无法确定的，在引入适当同步后，事件之间才有了 happens-before 关系。因此多线程的正确性也不能依赖任何一个线程的执行速度，不能通过原地等待(sleep)来假定其他线程的事件已经发生。
* 基本线程原语的选择：

线程的创建与等待(muduo::Thread)

mutex 的创建、销毁、加锁、解锁(muduo::MutexLock)

条件变量的创建、销毁、等待、通知、广播(muduo::Condition)

**更高层的封装**

mutex::ThreadPool，mutex::CountDownLatch

* 编写线程安全程序的一个难点在于**线程安全是不可组合的**，一个函数调用了两个线程安全的函数，这个函数本身很可能不是线程安全的。
* C++标准库容器和 std::string 都不是线程安全的，只是 std::allocator 保证是线程安全的。一方面是为了避免不必要的性能开销，另一方面就是单个函数的线程安全并不具备可组合性。
* 尽量使用相同的方式创建线程，且线程的创建最好能在初始化阶段全部完成。好处是线程不需要销毁，可以伴随进程一直运行，彻底避开线程安全退出可能面临的各种困难。
* 一个服务程序的线程数目应该于当前负载无关，而应该与机器的 CPU 数目有关；对实时性有要求的，线程数目不应该超过 CPU 数目，这样可以基本保证新任务总能及时得到执行。
* 每个线程应该又明确的职责，例如 IO 线程，计算线程等。且线程之间的交互应该尽量简单，理想情况下线程之间只用消息传递就；如果必须用锁，那么最好避免一个线程同时持有两把或更多的锁，避免产生死锁。



## Good 引用

### C++内存模型

https://www.zhihu.com/question/24301047

http://wilburding.github.io/blog/2013/04/07/c-plus-plus-11-atomic-and-memory-model/

### CPU Cache and Memory Ordering

 何登成

http://www.valleytalk.org/wp-content/uploads/2013/07/CPU-Cache-and-Memory-Ordering.pdf

* Thrashing

https://en.wikipedia.org/wiki/Thrashing_(computer_science)

​      In computer science, thrashing occurs when a computer's virtual memory subsystem is in a constant state of paging, rapidly exchanging data in memory for data on disk, to the exclusion of most application-level processing

* ​

## 基础知识

### __thread 变量

参考：https://gcc.gnu.org/onlinedocs/gcc/Thread-Local.html#Thread-Local

​      Thread-local storage (TLS) is a mechanism by which variables are allocated such that `there is one instance of the variable per extant thread`. 

​      The `__thread` specifier may be applied to any global, file-scoped static, function-scoped static, or static data member of a class. It may not be applied to block-scoped automatic or non-static data member.

#### pthread_key_t

参考：http://blog.csdn.net/yusiguyuan/article/details/21785641



### pthread condition variable

用于线程同步

#### 基本用法

```c++
#include <pthread.h>
int pthread_cond_init(pthread_cond_t *restrict cond,
                     pthread_condattr_t *restrict attr);
int pthrad_cond_destroy(pthread_cond_t *cond);

int pthread_cond_wait(pthread_cond_t *restrict cond,
                     phtread_mutex_t *restrict mutex);
int pthread_cond_timedwait(pthread_cond_t *restrict cond,
                          phtread_mutex_t *restrict mutex,
                          const struct timespec *restrict timeout);

int pthread_cond_signal(pthread_cond_t *cond);
int pthread_cond_broadcast(pthread_cond_t *cond);
```

```c
while(1) {
    pthread_mutex_lock(&mutex);
    while(some_data == NULL) { // predicate to acccount for spurious wakeups,would also 
                               // make it robust if there were several consumers
       pthread_cond_wait(&cond,&mutex); //atomically lock/unlock mutex
    }

    char *data = some_data;
    some_data = NULL;
    pthread_mutex_unlock(&mutex);
    handle(data);
}
```

​        传递给 phtread_cond_wait的互斥量对条件进行保护。调用者把锁住的互斥量传递给函数，函数自动把线程放到等待条件的线程列表上，对互斥量解锁。这样就关闭了条件检查和线程进入休眠状态等待条件改变这两个操作之间的竞态。当 phtread_cond_wait 返回时，互斥量再次被锁住。



#### 假唤醒

参考：https://en.wikipedia.org/wiki/Spurious_wakeup

https://sites.google.com/site/embeddedmonologue/home/mutual-exclusion-and-synchronization/spurious-wakeup-and-infinite-waiting-on-condvar

在多处理器机器上可能发生，多个线程同时被唤醒。



#### 问题

* 为什么需要一个 mutex

https://stackoverflow.com/questions/2763714/why-do-pthreads-condition-variable-functions-require-a-mutex

* ​

