# EffectiveC++

[TOC]

date: 2017-04-05

## 让自己习惯C++

### 条款1 视C++为一个语言联邦

C++由4个部分组成

+ part of C
+ Object-Oriented C++
+ Template C++
+ STL

### 条款2 尽量以const, enum, inline 替换 #define

**关联：条款3、条款30**

宁可以编译器替换预处理器

\#define可以用于定义常量、定义宏。But

\#define不重视作用域，不能用于定义class专属属性，不提供任何封装性，还是使用const，enum吧；

而对于宏，也会产生一些陷阱。通过template inline函数可以获得宏带来的效率，以及一般函数的所有可预料行为和类型安全性；

但\#ifdef / \#ifndef 控制编译方面还是不可替代的。

### 条款3 尽可能使用const

**关联：条款27**

编译器帮忙保证const表示的语义约束。

定义const成员函数，使”操作const对象“成为可能。

注意：两个成员函数如果只是常量性不同，是可以重载的。

```c++
const obj& operator[](std::size_t idx) const;

obj& operator[](std::size_t idx);

```

​        为了避免代码重复，可以将non-const版本调用const版本（反向就不一定合理了，因为const版本是绝对不会对象的逻辑状态的）。

### 条款4 确定对象在使用前已先被初始化





## 模板与泛型编程

### 条款41 了解隐式接口和编译期多态
