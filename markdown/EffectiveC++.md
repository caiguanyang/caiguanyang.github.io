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

读取未初始化的值会导致不明确的行为。

最佳处理办法：

+ 永远在使用对象之前先将它初始化；
+ 对于内置类型以外的任何对象，初始化责任落在了构造函数身上，所以要确保每个构造函数初始化对象的每个成员，考虑采用初始化列表；
+ 对于成员间初始化顺序有要求的，可以通过加强设计解决（设计模式）

初始化列表：

+ 如果成员变量是 const或者 references，一定要初始化，而不能赋值
+ 初始化列表相比构造函数内赋值操作，效率更高（默认构造函数、赋值构造函数 VS copy 构造函数）
+ class 成员变量总是以其声明次序被初始化

跨编译单元初始化问题：

+ C++对“定义于不同编译单元的 non-local  static 对象”的初始化顺序并没有明确定义

+ 函数内的 local static 对象会在“该函数被调用期间，首次遇上该对象之定义式”时被初始化。并且初始化过程是线程安全的

  (单例模式的常见实现方式)

+ 书中说【函数内含 static 对象，会使它们在多线程系统中带有不确定性】，但是目前 c++会保证初始化过程是线程安全的。为了避免不同编译器的处理办法不同，可以在程序的单线程启动阶段手动调用这些函数，完成 static 对象的首次初始化操作

## 构造、析构、赋值运算

### 条款5 了解 C++默默编写并调用哪些函数

编译器会暗自为 class 创建 default 构造函数、copy 构造函数、copy assignment 操作符，以及析构函数，但是如果你已声明，则编译器不会生成。另外也只有你未声明，且程序中有用到时才会生成。

### 条款6 若不想使用编译器自动生成的函数，就该明确拒绝

关联：条款32,39,7,55

单例模式的常见实现方式，参考如下代码：

```c++
class Uncopyable {
  protected:
  	Uncopyable() {}
  	~Uncopyable() {}
  private:
  	Uncopyable(const Uncopyable&);
  	Uncopyable& operator=(const Uncopyable&);
};

阻止 Test 对象被拷贝：
  class Test : private Uncopyable {
    ...
  };
```

**注意**

1）private 继承（参考 条款32，39）

2）析构函数不一定是 virtual（参考 条款7）

3）Boost 库也有个版本哈（参考条款55）

### 条款7 为多态基类声明 virtual 析构函数







## 模板与泛型编程

### 条款41 了解隐式接口和编译期多态
template及泛型编程的世界，与面向对象有根本上的不同，在此世界中显示接口和运行时多态仍然存在，但是重要性降低，反倒是隐式接口和编译器多态移到了前头。
编译期多态：类似于哪一个重载函数该被调用；
运行期多态：类似于哪一个virtual函数该被绑定；

### 条款42 了解typename的双重意义

C++并不总把class和typename视为等价，有时候你一定得使用typename:

```c++
template<typename C>				    // 可以使用typename或者class 
  void fun(const C& container,           // 不允许使用typename
          typename C::iterator iter);    // 一定要使用typename,告诉编译器iterator是C的类型成员，非数据成员
```

