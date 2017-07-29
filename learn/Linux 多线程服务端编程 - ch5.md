# 高效的多线程日志

[TOC]

Date: 2017-07-23

## 问题解决思路





## 知识点

* Log Everything All the Time

参考：http://highscalability.com/log-everything-all-time

* Dapper

参考：https://bigbully.github.io/Dapper-translation/



## 基础知识

### double buffering

参考：https://en.wikipedia.org/wiki/Multiple_buffering

​      In computer science, multiple buffering is the use of more than one buffer to hold a block of data, so that a "reader" will see a complete (though perhaps old) version of the data, rather than a partially updated version of the data being created by a "writer".

### swap

​       交换两个对象的值，当标准库中的 swap 算法(1次copy 构造，2次赋值构造)不能满足需求时，可以有自定义的实现，如采用 pimpl (pointer to implementation) 手法的对象。

**交换两个整数**

```c++
void swap(int a, int b) {
  // method - 1
  a ^= b;
  b ^= a;
  a ^= b;
  // method = 2
  a = a + b;
  b = a - b;
  a = a - b;
}
```

​       交换 vector 或者其他容器，如果两个对象采用相同的分配器，则只需要交换控制信息即可，不涉及到容器内元素的拷贝，效率比较高。（ All iterators, references and pointers remain valid for the swapped objects.）











