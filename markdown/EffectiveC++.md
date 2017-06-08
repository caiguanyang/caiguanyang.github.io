# EffectiveC++

[TOC]



**date: 2017-04-05**



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
  	const Uncopyable& operator=(const Uncopyable&);
};

阻止 Test 对象被拷贝：
  class Test : private Uncopyable {
    ...
  };
```

**注意**

1）private 继承（参考 条款32，39）

​	Test 的 member 函数或 friend 函数，以及任何对象，尝试拷贝 Test 对象时，编译器便尝试生成一个 copy 构造函数和一个 copy assignment 操作符，编译器生成版会尝试调用 base class 的对应兄弟，那些调用会被编译器拒绝，因为 base class 的拷贝函数是 private 

（private 成员，在任何继承模式下，对 derived class都是不可见的）

2）析构函数不一定是 virtual（参考 条款7）

3）Boost 库也有个版本哈

### 条款7 为多态基类声明 virtual 析构函数

+ 带有多态性质的 base classes 应该声明一个 virtual 析构函数。如果 class 带有任何 virtual 函数，就应该拥有一个 virtual 析构函数
+ classes 设计的目的如果不是作为 base classes 使用(如 STL)，或不是为了具备多态性，就不应该声明virtual 函数（请不要继承标准容器，或者任何其他“带有 non-virtual析构函数”）

​       析构函数的运作方式是，最深层派生的哪个 class，其析构函数最先被调用，然后是其每一个 base class 的析构函数被调用。（编译器会在 derived class 的修改函数中调用 base classes 的析构函数，如果找不到实现，就会报链接错误）

​	C++明确指出，当 derived class 对象经由一个 base class 指针删除，而该 base class 带有一个 non-virtual 析构函数，其结果未有定义—实际执行时通常发生的是对象的 derived 成分没被销毁，造成一个诡异的“局部销毁”对象。

### 条款8 别让异常逃离析构函数

危害：析构函数抛出异常，且未捕获，可能导致不明确的行为，或者程序结束执行

+ 析构函数绝对不要吐出异常，如果析构函数调用的函数可能抛出异常，则它应该捕获异常，然后吞下他们或结束程序；
+ 当然也可以定义一个函数让客户手动调用，析构函数中也调用，做到双保险。

**构造函数抛出异常，将导致析构函数不被执行**



### 条款9 绝不在构造和析构过程中调用 virtual 函数

危害：导致非预期结果

+ 在 derived class 对象的 base class 构造期间，对象的类型是 base class，而不是 derived class。

构造过程：先 base class 构造，然后是 derived class

析构过程：先 derived class 析构，然后是 base class

调用虚函数的话，指针应该指向derived class 的函数，而函数调用的一般的都是 local变量，所以构造过程和析构过程，那些 local 变量要么未初始化，要么已经被释放，所以调用他们会发生不确定行为。



### 条款10 令 operator= 返回一个 reference to *this

为了实现连锁赋值（x=y=z=15），赋值操作符必须返回一个 reference 指向操作符的左侧实参。

```c++
class Widget {
  public:
  Widget& operator=(const Widget& rhs) {
    ...
    return *this;
  }
};
```

适用与所有赋值操作符重载，如 += -=等，只是一个协议，并无强制性。

### 条款11 在 operator= 中处理“自我赋值”

要考虑自我赋值和异常安全性。

```c++
Widget& Widget::operator=(const Widget& rhs) {
  Bitmap* pOrig = pb;
  pb = new Bitmap(*rhs.pb);   // 备份原指针，避免构造 Bitmap 的过程中出错，导致原指针不可用
  delete pOrig;  			 // 删除原指针
  return *this;
}
```

### 条款12 复制对象时勿忘其每一个成分

+ 如果自己定义 copy 函数，那么当类的成员变量做变更时 ，构造函数，copy 函数，析构函数都需要做相应的调整，避免遗漏；
+ derived class 的 copy 函数不能忘记调用 base class 的 copy 函数完成复制





## 资源管理

### 条款13 以对象管理资源

​       为什么呢？因为如果自己管理资源的话，很容易“忘记”执行 delete 操作，释放动态分配的资源。如在执行 delete 之前程序已经返回（for 循环中的return，其他代码块的提前 return，导致执行不到 delete）

​       将资源放进对象内，我们便可依赖 C++的“析构函数自动调用机制”确保资源被释放。

+ 获得资源后立刻放进管理对象内。资源取得时机便是初始化时机（RAII: Resource Acquisitions Is Initialization）：在构造的时候获取资源，在析构的时候释放资源

对于 heap-based 类型资源，可以关注下智能指针：std::auto_ptr, std::tr1::shared_ptr

其他类型的资源，就需要自己实现管理类了，参考条款14

### 条款14 在资源管理类中小心 coping 行为

问题点：当一个 RAII 对象被复制，会发生什么事情？

常见策略：

+ 禁止复制。不能复制的，要明确制止
+ 对底层资源寄出“引用计数法”。参考 tr1::shared_ptr
+ 转移底部资源的拥有权。参考 std::auto_ptr

C++11 和 Boost 库中都有相应解决方案

### 条款15 在资源管理类中提供对原始资源的访问





### 条款16 成对使用 new 和 delete 时要采用相同形式

p103



### 条款17 以独立语句将 newed 对象置入智能指针

p105



## 设计与声明

p108



### 条款20 宁以 pass-by-referencee-to-const替换 pass-by-value

p116

缺省情况下 C++以 by value 方式传递对象至函数；

值传递的弊端：

1）C++对象进行值传递时，有时是比较昂贵的操作，涉及到成员对象的多次构造和析构；

​       （例外：内置类型，STL 的迭代器和函数对象）

2）继承体系中的对象值传递是，可能会出现 slicing(对象切割)问题。如函数的参数类型为基类，但是实际传递的值为子类的实例，会导致函数中只能使用基类中定义的行为。



### 条款21 必须返回对象时，别妄想返回其 reference

p120

绝不要返回 pointer或 reference 指向一个 local stack 对象，或返回 reference 指向一个 heap-allocated 对象（需要依赖调用方去释放空间，又会引发其他问题），或返回 pointer 或 reference 指向 local static 对象而有可能同时需要多个这样的对象。

参考：条款4





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

