---
layout: post
title: C++返回值优化的小试验
date: 2021-06-30
Author: levy5307
tags: []
comments: true
---

之前知道C++有返回值优化，但是一直有点不太敢用，生怕用不好就会导致性能问题。最近本着刨根问底的态度，亲自写代码试验了一下。代码如下：

```
class Str
{
private:
    int size;
public:
    Str() { std::cout << "调用无参构造函数" << std::endl; }
    Str(int n) :size(n) { std::cout << "调用有参构造函数" << std::endl; }
    Str(const Str& s) :size(s.size) { std::cout << "调用拷贝构造函数" << std::endl; }
    Str(Str&& ss) :size(ss.size) { std::cout << "调用移动构造函数" << std::endl; }
    ~Str()
    {
        std::cout << "调用析构函数" << std::endl;
    }
};

Str get_object()
{
    Str object(3);
    return object;
}
int main()
{
    Str target = get_object();
    return 0;
}
```

对于没有优化的代码，其打印的日志应该如下所示：
```
调用有参构造函数		// 构造局部对象object
调用移动构造函数		// 用局部对象object构造局部临时对象（get_object函数的return处）
调用析构函数			// 析构局部对象object
调用移动构造函数		// 用临时对象构造target
调用析构函数			// 析构临时对象
调用析构函数			// 析构target
```

然而实际执行起来，打印的日志如下：
```
调用有参构造函数
调用析构函数
```

这说明返回值优化起了作用，原本应该有将一系列的构造和析构，优化成了只有target对象的构造和析构。优化效果还是很明显的。

### Reference

[RVO and NRVO in cpp](https://www.codenong.com/cs106863732/)


