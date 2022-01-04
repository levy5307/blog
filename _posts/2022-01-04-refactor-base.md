---
layout: post
title: 谈谈代码重构
date: 2022-01-04
Author: levy5307
tags: []
comments: true
toc: true
---

过去几年里做了很多重构相关的工作，所以一直想写一篇关于代码重构的文章。但是国内互联网公司环境下，重构工作其实是不太被老板们重视的。因为不管重构做不做，功能一直都是有的，重构本身并没有带来什么比较亮眼的功能点，反而容易带来bug（除非重构工作带来很多性能的提升，当然绝大多数重构工作并不会带来这种功效）。所以一度对于重构这一块是很灰心的，颇有种老顽童想要拼命忘记九阴真经的感觉，因此文章也一直放着没写。现在我想明白了，人嘛，更重要的是要为自己负责，所以看到不顺眼的代码还是会去改，而且代码重构本身是非常有意义的。比如之前Pegasus的load balance模块，那块真的是太乱了，如果不重构后面完全无法添加新的负载均衡策略。

其实提到重构，不得不说的就是设计模式。之前有同事跟我讲：设计模式这个东西没什么用，因为很少有机会能够套用上这些所谓的模式。我个人认为，设计模式这个东西本来就不是用来生搬硬套的，学习设计模式更多的是学习其中的思想和精髓，就如同武侠小说中的武功心法。比如郭靖常年研习九阴真经里的内功心法，并将融会贯入到降龙十八掌里，使其刚中有柔，因此在大战金轮法王+蒙古三杰时能够愈战愈勇并逐渐占据上风。与之乔峰在少室山中一味刚猛而无法久战相比，自然是高明的多了。我本人在工作中也经常遇到一个场景，即在重构某部分代码时，压根就没有想过去用什么设计模式。等设计完、画完类图之后才恍然大悟，原来无形中已经用上了设计模式。

## 单例模式

单例模式是一个非常常用的设计模式，主要用于一个类只有一个对象的场景。它有两种实现方式：

- 饿汉式

懒汉式是指，对于该类的单例对象，不管日后是否使用，都事先创建一个对象，其代码如下：

```
class Singleton
{
public:
    static Singleton& instance()
    {
	return instance;
    }

private:
    Singleton() = default;
    Singleton(const singleton &) = delete;
    Singleton &operator=(const singleton &) = delete;

    private Singleton instance;
};
```

该方式的缺点在于，对于某个类，即使永远没有调用过`instance()`函数，该单例对象也早已创建完，造成资源的浪费。

其优点也很明显，即线程安全。

- 懒汉式

懒汉式是指，对于该类的单例对象，只有在获取的时候才去创建，其常见写法如下：

```
class Singleton
{
public:
    static Singleton *instance()
    {
	if (nullptr == instance) {
	   instance.reset(new Singleton());
	}
	return instance.get();
    }

private:
    Singleton() = default;
    Singleton(const singleton &) = delete;
    Singleton &operator=(const singleton &) = delete;

    private std::unique_ptr<Singleton> instance;
};
```

这样做的好处是，当某个类的`instance()`从没有调用过时，便可以不用创建该类的对象，减少不必要的资源浪费。其缺点也很明显，该方式不是线程安全的，原因在于instance()函数的`if (nullptr == instance)`这一行。

对于非线程安全这一点，可以通过以下几种方法来解决：

1. 加锁。对if语句进行加锁可以实现线程安全，但是这样对性能的开销很大

2. 利用"C++11中静态变量的初始化时线程安全的"这一条特性。具体实现代码如下：

```
class Singleton
{
public:
    static Singleton& instance()
    {
	static Singleton instance;
	return instance;
    }

private:
    Singleton() = default;
    Singleton(const singleton &) = delete;
    Singleton &operator=(const singleton &) = delete;
};
```

上述实现集合了不浪费资源、又线程安全两大优点。

另外，仔细看上面的代码，不管是懒汉式还是饿汉式，都使构造函数对外不可见。这是为了防止有代码直接`new Singleton`，从而破坏了一个类只有一个对象的约定。

## 工厂模式

