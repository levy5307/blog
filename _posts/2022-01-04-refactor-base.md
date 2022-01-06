---
layout: post
title: 谈谈代码重构
date: 2022-01-04
Author: levy5307
tags: []
comments: true
toc: true
---

过去几年里做了很多重构相关的工作，所以一直想写一篇关于代码重构的文章。但是国内互联网公司环境下，重构工作其实是不太被老板们重视的。因为不管重构做不做，功能一直都是有的，重构本身并没有带来什么比较亮眼的功能点，反而容易带来bug（除非重构工作带来很多性能的提升，当然绝大多数重构工作并不会带来这种功效）。所以一度对于重构这一块是很灰心的，颇有种老顽童想要拼命忘记九阴真经的感觉，因此文章也一直放着没写。现在我想明白了，人嘛，更重要的是要为自己负责，所以看到不顺眼的代码还是会去改。而且代码重构本身是非常有意义的，比如之前Pegasus的load balance模块，那块真的是太乱了，如果不重构后面完全无法添加新的负载均衡策略。

所以下面主要针对本人的重构经验进行总结和反思。

## 设计模式

其实提到重构，不得不说的就是设计模式。之前有同事跟我讲：设计模式这个东西没什么用，因为很少有机会能够套用上这些所谓的模式。我个人认为，设计模式这个东西本来就不是用来生搬硬套的，学习设计模式更多的是学习其中的思想和精髓，就如同武侠小说中的武功心法。比如郭靖常年研习九阴真经里的内功心法，并融入到降龙十八掌里，使其刚中有柔，因此在大战金轮法王+蒙古三杰时能够愈战愈勇并逐渐占据上风。与乔峰在少室山中一味刚猛而无法久战相比，自然是高明的多了。我在工作中也经常遇到一个场景，即在重构某部分代码时，压根就没有想过去用什么设计模式。等设计完、画完类图之后才恍然大悟，原来早已在无形中用上了。

### 单例模式

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

### 工厂模式

工厂模式也是普遍采用的一种设计模式，其优点主要有如下几个：

- 解耦。假如class A要获取Class B的各种不同子类的对象，只需要通过传参给工厂，工厂根据参数返回具体的子类对象，并使用Class B类型指针返回。此时Class A无需知道Class B的子类情况，达到了屏蔽的效果。

- 降低代码重复。如果创建对象的过程很复杂，需要大量的代码，那么将创建对象的过程封装到工厂中，可以显著减少重复代码。

#### 简单工厂

最常用、也最简单的工厂模式就是简单工厂模式，其类图如下所示：

![](../images/simple-factory1.png)

上图中的Product只有一种，因此类图很简单。当Product有多种，且是固定数量时，同样也可以使用简单工厂模式。只不过是Factory类针对不同的产品，提供不同的接口。

![](../images/simple-factory2.png)

那么当Product数量不固定时，简单工厂还能满足需求吗？

答案是否定的。因为由于Product数量不固定，所以每当增加一个Product时，都需要侵入式修改Factory代码，非常不优雅。

#### 工厂方法模式

对于Product数量不固定的这种情况，需要使用工厂方法模式。其思路也很简单，就是避免侵入式修改。那么避免侵入式修改的最简单的方法就是，通过增加工厂子类来应对增加的Product。其类图如下：

![](../images/factory-method.png)

如图中所示，Factory是一个接口类，其有不同的子类，每个子类对应一个具体的Product。当增加Product时，只需要增加Factory的子类即可。

#### 抽象工厂模式

有时产品会有两个维度，例如猫狗老鼠、公和母。此时使用简单工厂模式和工厂方法模式都不能满足需求，此时就需要使用抽象工厂模式了。

其原理在于，对于产品的两个维度，首先区分出哪个容易变化、哪个不容易变化。如前面举得例子，公和母是不会发生改变的，自然界中只有这么两种可能。然而动物种类则是可能发生变化的，最开始系统中只有猫狗和老鼠，后面可能会增加兔子、鸵鸟等等。

其次，对于选取出的不易变化变化的维度，采用简单工厂中的方法，即增加创建函数的方法；而对于容易发生变化的维度，前面讲过，如果采用增加创建函数的方法的话，很容易带来侵入式修改，因此需要采用工厂方法中的方式，即增加子类的方式。

具体类图如下：

![](../images/abstract-factory.png)

### Builder模式

Builder模式也是一种创建型设计模式，其主要应用场景有如下两个：

- 类的构造函数参数特别多，其中一部分是必要的，另外还有很多参数是可选的，这样会导致需要创建很多个构造函数。例如：

```
class Example {
public:
    Example(int a);
    Example(int a, int optional1);
    Example(int a, std::string& optional2);
    Example(int a, int optional1, std::string& optional2);

private:
    int a;
    int optional1;
    std::string optional2;
};
```

在上面的例子中，有1个必要参数，2个可选参数，导致构造函数需要创建4个。如果参数数量再增多的话，其复杂程度可想而知。因此需要采用Builder模式，其大概实现如下：

```
class Example {
public:
    Example(int a, int optional1, std::string& optional2);

    class Builder {
    public:
        Builder(int a);

        Builder& setOptional1(int optional1);
        Builder& setOptional2(std::string optional2);
        Example* build() {
            return Example(a, optional1, optional2);
        }

    private:
        int a;
        int optional1;
        std::string optional2;
    };

private:
    int a;
    int optional1;
    std::string optional2;
};
```

这里有很多同学会问，就只使用必选参数来实现一个构造函数`Example(int a)`，其他使用set函数来实现不可以吗？

当然是不可以的，这里有两个原因：

1. 这些参数有可能是const类型的，不支持set函数

2. 即使支持set函数，当调用完`new Example(int a)`之后，会获取一个“半成品” 对象。这样需要用户代码逻辑来保证不会错误的使用这种“半成品”对象，导致系统不够健壮的。

- 对象的创建，需要对给出的参数进行合法检查，当检查失败时则不进行创建。 

这种情况可以参考本人之前做Pegasus load balance重构时的[实现](https://github.com/XiaoMi/rdsn/pull/908/files?diff=unified&w=0#diff-c013463d4991b89fb5d5328eb8894552e0c655d1973fa800a0bb2a5e9d347c0aR98)

```
std::unique_ptr<ford_fulkerson> build()
{
    // do some caculate
    ...

    if (0 == higher_count && 0 == lower_count) {
        return nullptr;
    }
    return dsn::make_unique<ford_fulkerson>(
            _app, _nodes, _address_id, higher_count, lower_count, replicas_low);
}
```

在build()函数中对一些参数进行了校验，当校验失败时不进行创建。而这些校验无法在构造函数中进行。很显然，在这种情况下使用set函数也是很不合理的，因为这会导致一些本不应该创建的对象，被作为半成品被创建出来。

## Specific Way In C++ 

## Others

