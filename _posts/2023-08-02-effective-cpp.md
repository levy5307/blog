---
layout: post
title: Effective C++复习笔记
date: 2023-08-02
Author: levy5307
tags: []
comments: true
toc: true
---

## 条款01：视C++为一个语言联邦

C++主要由4个次语言组成：

- C。C++仍是以C为基础

- Object-Oriented C++。这部分就是C with Classes

- Template C++。这部分是C++的模板编程

- STL

需要注意的是，每个部分的高效编程守则可能不太一样。例如：

- 对于C-like类型，pass-by-value比pass-by-reference更高效

- 对Object-Oriented C++部分，则pass-by-reference则会更高效

- Template C++更需要使用pass-by-reference，因为有时候你都不知道你在处理什么类型的对象

- STL部分，迭代器和函数对象都是在C指针上构造出来的，则使用pass-by-value会更高效。

## 条款02：尽量以`const`, `enum`, `inline`替换`#define`

这个条款或许可以称为“宁可以编译期替换预处理器”。

对于常数定义，使用const替换宏定义。宏定义有如下几个问题：

- 宏定义部分会由预处理器对其进行替换处理，其从来不会被编译器看见。因此该部分所使用的部分并不会进入符号表，这样进行调试时会比较困惑。例如：

  `#define ASPECT_RATIO 1.653`中`ASPECT_RATIO`并不会进入符号表，因此当出现编译错误时，错误信息可能是1.63，而不是`ASPECT_RATIO`

- 宏定义并不重视作用域。一旦宏被定义，它就在其后的编译过程中有效。这意味着`#define`不仅不能够用来定义class专属常量，也不能提供任何封装性。

- 宏定义没有类型信息，这样编译器便没办法给出一些类型安全检查

使用const做常量替换，却没有上述几个问题，例如：

  `const double AspectRatio = 1.653   // 可以进入符号表，便于排查问题。并且编译期可以进行类型安全检查`

  ```cpp
  class GamePlayer {
  private:
     static const int NumTurns = 3; // 不仅可以进入符号表及安全检查，还可以限定作用域
  }
  ```

`#define`定义宏的主要问题是非常容易出错，例如：

```cpp
#define MULTIPLY(a, b) a*b;
```

对于上述宏，当采用如下代码调用时，便是有问题的：

```
MULTIPLY(1+2, 3+4)
```

其最后获取的结果是：`1+2*3+4`

因此宏的定义通常需要为宏中的所有参数加上小括号：

```cpp
#define MULTIPLY(a, b) (a)*(b);
```

但是这样并不能解决所有问题，例如：

```cpp
#define CALL_WITH_MAX(a, b) f((a) > (b) ? (a) : (b))
```

当采用如下代码调用时，则产生不可思议的后果：

```cpp
int a = 5, b = 0;
CALL_WITH_MAX(++a, b)       // a被累加2次
CALL_WITH_MAX(++a, b+10)    // a被累加1次
```

而采用inline函数替换宏，完全可以杜绝这些问题：

```cpp
// 由于采用了模板函数实现，因此可以支持很多类型
template<typename T>
inline T callWithMax(const T& a, const T& b) {
   return a > b ? a : b;
}
```

另外书中还讲到，当编译期不允许static整数型class常量的in class初值设定时（例如`GamePlayer`例子中的static class常量），可以使用the enum hack做法来实现。但是当前主流的编译器都支持了，所以这里不再讲解。

## 条款03：尽可能使用const

- const指针

通过const可以指定指针自身、指针所指物为const。具体含义可以将*翻译成point to，并从后向前读，例如：

`const char* p`表示p point to const char，即non-const pointer, const data

`char* const p`表示const p point to char，即const pointer non-const data

- const返回值

考虑有理数的operator*声明式：

```
class Rational { ... };
const Rational operator* (const Rational &lhs, const Rational &rhs);
```

operator*函数的返回值是const对象，这样的情况主要是为了避免如下这种误操作：

```
Rational a, b, c;
...
if (a * b = c)
```

本身代码是想做比较操作，但是误写成了赋值。如果函数返回值是const对象，编译器将会报错，这一点对于内置类型也是这样的。因此引申出了一个规则：良好的用户自定义类型的特征是避免和内置类型不兼容 。

- const成员函数

const实施于成员函数主要有两个理由：

- 它们使class接口比较容易被理解。添加了const的函数就可以比较明显的告诉用户，该函数不会更改对象的内容。

- 他们使操作const对象成为可能。非const成员函数无法被const对象调用，因为该函数有可能会更改对象内容。

