# Blocking-Queue

[TOC]

## 介绍

作为多线程编程的利器，可以用于实现任务队列，生产者消费者队列等



## 实现

### muduo 库

#### BlockingQueue

```c++
Condition _notEmpty;
MuthexLock _mutex;

push:
    _mutex.lock();
      
pop:
    _mutex.lock();
    _notEmpty.wait();
```





#### BoundedBlockingQueue

```c++
Condition _notEmpty;
Condition _notFull;
MuthexLock _mutex;

push:
    _mutex.lock();
    _notFull.wait();
    _notEmpty.notify();

pop:
    _mutex.lock();
    _notEmpty.wait();
    _notFull.notify();

```

参考 boost::circualar_buffer

https://theboostcpplibraries.com/boost.circularbuffer



### tbb concurrent_queue

tbb 库高性能的秘密：https://software.intel.com/zh-cn/blogs/2009/08/13/tbb-concurrent_queue/

​        Herb Sutter在DDJ 一文中抛出并行编程的三个简单论点，一是分离任务，使用更细粒度的锁或者无锁编程；二是尽量通过并行任务使用CPU资源，以提高系统吞吐量及扩展性；三是保证对共享资源访问的一致性。

​        concurrent_queue 采用多个 micro_queue 提高并行度；

​       

```c++
void internal_push( const void* src, item_constructor_t construct_item ) {
         concurrent_queue_rep<T>& r = *my_rep;
         ticket k = r.tail_counter++;
         r.choose(k).push( src, k, *this, construct_item );
}

//! Approximately n_queue/golden ratio 
//  将n_queue进行黄金分隔的话，为5、3 (n_queue - phi) : phi = ratio(1.618)
//  phi=3或者5都可以，最终k*phi%n_queue的离散索引是一致的  0 3 6 1 4 7 2 5 0  
static const size_t phi = 3;
// must be power of 2
static const size_t n_queue = 8;

template<typename T>
struct concurrent_queue_rep : public concurrent_queue_rep_base {
    micro_queue<T> array[n_queue];

    //! Map ticket to an array index
    static size_t index( ticket k ) {
        // ??? k*phi 有什么好处呢？  0 3 6 1 4 7 2 5 离散
        return k*phi%n_queue;
    }

    micro_queue<T>& choose( ticket k ) {
        // The formula here approximates LRU in a cache-oblivious way.
        return array[index(k)];
    }
};
```



