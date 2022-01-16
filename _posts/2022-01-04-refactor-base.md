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

答案是否定的。因为由于Product数量不固定，所以每当增加一个Product时，都需要侵入式修改Factory代码（不符合开闭原则），非常不优雅。

#### 工厂方法模式

对于Product数量不固定的这种情况，需要使用工厂方法模式。其思路也很简单，就是避免侵入式修改（符合开闭原则）。那么避免侵入式修改的最简单的方法就是，通过增加工厂子类来应对增加的Product。其类图如下：

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

- 类的构造函数参数特别多，其中一部分是必要的，另外还有很多参数是可选的。这样会导致需要创建很多个构造函数。例如：

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

- 对象的创建需要对给出的参数进行合法性检查，当检查失败时不进行创建。 

这种情况可以参考本人之前做[Pegasus load balance重构时的实现](https://github.com/XiaoMi/rdsn/pull/908/files?diff=unified&w=0#diff-c013463d4991b89fb5d5328eb8894552e0c655d1973fa800a0bb2a5e9d347c0aR98)

其大致代码如下：

```
std::unique_ptr<ford_fulkerson> build()
{
    // do some caculate
    ...

    if (0 == higher_count && 0 == lower_count) {
        return nullptr;
    }
    return dsn::make_unique<ford_fulkerson>(higher_count, lower_count);
}
```

如上所示，在`build()`函数中对一些参数进行了校验，当校验失败时不进行创建。而这些校验无法在构造函数中进行。很显然，在这种情况下使用set函数也是很不合理的，因为这会导致一些本不应该创建的对象，作为“半成品”被创建出来。

### 原型模式

最近在读西游记，里面有一个片段让人印象深刻：

> 拔一把毫毛，丢在口中嚼碎，望空中喷去，叫一声“变”，即变做三二百个小猴，周围攒簇。

原型模式就有类似的功效，即克隆复制。下面为代码示例：

```
class Monkey {
public:
    Monkey(uint32_t height, uint32_t weight) {
        this->height = height;
        this->weight = weight;
    }
    Monkey(const Monkey& monkey) {
        this->height = monkey.height;
        this->weight = monkey.weight;
    }

    Monkey* clone() {
        return new Monkey(*this);
    }

private:
    uint32_t height;
    uint32_t weight;
};
```

通过调用`monkey->clone()`，即可复制出与monkey一模一样的小猴子:

```
void doBussiness(Monkey *monkey) {
    auto monkey1 = new Monkey(180, 80);
    auto monkey2 = monkey1->clone();
    auto monkey3 = monkey1->clone();
}
```

可能有人会问，直接用new创建不行吗？比如下面这样：

```
void doBussiness(Monkey *monkey) {
    auto monkey1 = new Monkey(180, 80);
    auto monkey2 = new Monkey(180, 80);
    auto monkey3 = new Monkey(180, 80);
}
```

这样当然是不好的，因为有大量的重复代码，重复代码往往意味着bad smell，因为一旦修改，就要修改很多处地方，非常不优雅。

另外clone函数最终调用的也是拷贝构造函数，那直接使用拷贝构造函数不可以吗？

当然是不可以的。有两个原因：

1. 因为clone函数可以实现多态，而拷贝构造函数不可以。使用多态可以自动识别出其具体类型，调用实际子类的clone函数 

2. 使用拷贝构造函数需要知道具体类的类型，这样会带来耦合（不符合开闭原则）。比如Monkey有多个子类，需要知道其具体是属于哪个子类，然后去调用其拷贝构造函数，带来了耦合性。

举个例子：

猴子其实是一个大类，具体可以分为猕猴、金丝猴等等。

```
class Monkey {
public:
    Monkey(uint32_t height, uint32_t weight) {
        this->height = height;
        this->weight = weight;
    }
    Monkey(const Monkey& monkey) {
        this->height = monkey.height;
        this->weight = monkey.weight;
    }
    virtual ~Monkey() = 0;

    virtual std::unique_ptr<Monkey> clone() = 0;

protected:
    uint32_t height;
    uint32_t weight;
};
```

猕猴有一个特性，即有些猕猴是没有尾巴的，所以其多一个参数`hasTail`

```
class Macaque : public Monkey {
public:
    Macaque(uint32_t height, uint32_t weight, bool hasTail)
    : Monkey(height, weight) {
        this->hasTail = hasTail;
    }
    Macaque(const Macaque& monkey) : Monkey(monkey) {
        this->hasTail = monkey.hasTail;
    }

    std::unique_ptr<Monkey> clone() {
	return std::make_unique<Macaque>(*this);
    }

private:
    bool hasTail;
};
```

而金丝猴在某些公司特指比较重要的员工，一般都是领导层，所以有一个参数`officialRank`

```
class GoldenMonkey : public Monkey {
public:
    GoldenMonkey(uint32_t height, uint32_t weight, uint16_t officalRank)
    : Monkey(height, weight) {
        this->officalRank = officalRank;
    }
    GoldenMonkey(const GoldenMonkey& monkey) : Monkey(monkey) {
        this->officalRank = monkey.officalRank;
    }

    std::unique_ptr<Monkey> clone() {
	return std::make_unique<GoldenMonkey>(*this);
    }

private:
    uint16_t officalRank;
};
```

当我们使用拷贝构造函数进行复制时： 

```
void doBussiness(Monkey *monkey) { 		// monkey实际是猕猴 
    // How to copy? 
    // 1. 根本不知道monkey的类型，不知道该调GoldenMonkey拷贝构造函数还是Macaque的拷贝构造函数。
    // 2. 即使知道是猕猴，也会带来对Macque的耦合
}
```

而使用clone函数就比较简单了，它支持多态，可以自动根据实际类型来调用具体子类的clone函数，并且完全不会带来耦合（符合开闭原则）： 

```
void doBussiness(Monkey *monkey) {
    auto monkey1 = monkey->clone();
    auto monkey2 = monkey->clone();
    auto monkey3 = monkey->clone();
}
```

### 享元模式

享元，故名思议，就是分享元数据的意思。在讲享元模式前，还是先举一个西游记中的例子：

> 玉帝大恼，即差四大天王，协同李天王并哪吒太子，点二十八宿、九曜星官、十二元辰、五方揭谛、四值功曹、东西星斗、南北二神、五岳四渎、普天星相，共十万天兵，布一十八架天罗地网，下界去花果山围困，定捉获那厮处治

这里的十万天兵天将，不是先从老百姓中征兵、训练，然后再出征，等出征完后各回各家各找各妈，因为这样效率实在太低了。而是事先准备好的军队，等用的时候直接从军队中调遣。正所谓养兵千日用兵一时。

这就是享元模式的基本思想。下面看一下具体的代码示例：

```
class Solider {
public:
    Solider(std::string name, std::string desc) {
        this->name = name;
        this->desc = desc;
    }
    virtual ~Solider() = 0;

    virtual void fight() = 0;
    const std::string& getName() const {
        return this->name;
    }

private:
    std::string name;
    std::string desc;
};

// 天王
class HeavenlyKing : public Solider {
public:
    HeavenlyKing(std::string name, std::string desc) : Solider(name, desc) {}

    void fight() {
        // 天王战斗逻辑
    }
};

// 哪吒
class Nezha : public Solider {
public:
    Nezha(std::string name, std::string desc) : Solider(name, desc) {}

    void fight() {
        // 哪吒战斗逻辑
    }
};

```
上面代码中实现了一个Solider的虚基类，其有天王和哪吒等几个子类。

```
class Army {
public:
    Army() {
        Solider *liudehua = new HeavenlyKing("刘德华", "德艺双馨");
        soliders[liudehua->getName()].reset(liudehua);

        Solider *zhangxueyou = new HeavenlyKing("张学友", "歌神");
        soliders[zhangxueyou->getName()].reset(zhangxueyou);

        Solider *nezha = new Nezha("哪吒", "三太子");
        soliders[nezha->getName()].reset(nezha);
    }

    Solider* getSolider(const std::string &name) {
        const auto &iter = soliders.find(name);
        if (iter != soliders.end()) {
            return iter->second.get();
        } else {
            return nullptr;
        }
    }

private:
    std::unordered_map<std::string, std::unique_ptr<Solider>> soliders;
};
```

上面创建了一个Army对象，代表军队。这里使用了享元模式，预先创建出了天王、哪吒等对象，放入其成员变量soliders中。当需要使用solider时，可以通过`getSolider`来获取具体的对象，而无需现场创建。

### 装饰器模式

装饰器模式主要用于现有的类的包装，向一个现有的对象添加新的功能，同时又不改变其结构、不作侵入式修改（符合开闭原则）。

下面还是以西游记为例，介绍一下Decorator的作用：

```
class Team {
public:
    virtual ~Team() = 0;

    // 念经
    virtual void chantScriptures() = 0;
    // 打妖怪
    virtual void killMonsters() = 0;
};

class Tangseng : public Team {
public:
    void chantScriptures() {
	// 打坐
        sit();
	// 念经
        sing();
    }
    void killMonsters() {
        // do nothing
    }

private:
    void sit();
    void sing();
};
```

最开始，西游取经队伍只有唐僧一人。唐僧路上需要不断念经来提高佛学造诣，但是有个问题是，他不会杀怪，所以总是有各种妖怪来骚扰，很难静下心来学习念经。

后续随着剧情推进，孙悟空出现了，这货法力高强可以杀怪。唐僧正好欠缺这部分能力，需要孙悟空来增强。这里，Decorator可以发挥作用了：

```
class Sunwukong : public Team {
public:
    Sunwukong(std::unique_ptr<Team> team) {
        this->team = std::move(team);
    }

    void chantScriptures() {
        team->chantScriptures();
    }

    void killMonsters() {
        // kill monsters easily
    }

private:
    std::unique_ptr<Team> team;
};
```

当唐僧需要念经时，先命令孙悟空扫清妖怪，然后便可以安心打坐念经了：

```
auto tangseng = std::make_unique<Team>();
auto sunwukong = std::make_unique<Team>(std::move(tangseng));
sunwukong->killMonsters();
sunwukong->chantScriptures();
```

如下所示为相关类图：

![](../images/decorator-model.jpg)

可能有人会问，可不可以让唐僧自己获取打妖怪的能力？当然是不可以的，原因如下：

1. 唐僧就是一个和尚，就是念经用的，一个和尚打打杀杀像什么话。（单一职责）

2. 如果唐僧去完真经，李世民没玩够，又派了唐僧2号再去取一遍，那唐僧2号是不是也要修习杀妖怪的本领？显然这就是重复实现了。Decorator可以将一些功能独立抽象出来。

另外，我在设计[重构Pegasus负载均衡时](https://levy5307.github.io/blog/load-balance-refactor/)，也考虑过使用Decorator模式。

### 代理模式

唐王李世民想要去西天取经，以普度众生。但现实中有两个问题：

1. 李世民身为一国之君，肯定不能亲自去啊，毕竟还有国家需要治理，而且去西天取经也要经历九九八十一难，实在没有精力（单一职责）。

2. 李世民又不是佛教众人，人家如来佛也不愿意给你（隐藏被代理对象）

所以最终李世民选择唐僧代替他去西天取经。这就是一个典型的代理模式，而唐僧就是李世民西天取经的代理。

实例代码如下：

```
class BookKeeper {
public:
    virtual ~BookKeeper() = 0;

    virtual void getBook() = 0;
};

class WestHeaven : public BookKeeper {
public:
    void getBook() {
        // 发放真经
    }
};

class TangSeng : public BookKeeper {
public:
    TangSeng() {
        keeper.reset(new WestHeaven);
    }

    void getBook() {
        walkToWest();
        keeper->getBook();
        goBack();
    }

private:
    void walkToWest();
    void goBack();

    std::unique_ptr<BookKeeper> keeper;
};
```

而李世民，只要在想去求取真经的时候，委派代理（也就是唐僧）去就可以了。

```
class Lishimin {
public:
    Lishimin() {
        keeper.reset(new TangSeng);
    }
    void getBook() {
        keeper->getBook();
    }

private:
    std::unique_ptr<BookKeeper> keeper;
};
```

类图如下所示：

![](../images/proxy-model.jpg)

### 装饰器模式 vs 代理模式

有很多人分不清装饰器模式和代理模式的区别，这里说明一下：

- 其实通过前面讲的西天取经的例子就很容易分辨了：对于装饰器模式，是孙悟空和唐僧一起去西天取经，以增强其杀怪能力，强调的是“跟我来”；而代理模式，是李世民将西天取经的事情委托给了唐僧，强调的是“给我冲”。即装饰器模式关注于在一个对象上动态的添加功能与方法，然而代理模式关注于控制对对象的访问。换句话说，用代理模式，代理类（proxy class）可以对它的客户隐藏一个对象的具体信息（例如唐僧对李世民隐藏了西天取经的具体细节）。

- 当使用代理模式的时候，我们常常在一个代理类中创建一个对象的实例（代理模式的Class Tangseng），毕竟代理类代理的是谁自己应该知道。而当我们使用装饰器模式的时候，我们通常的做法是将原始对象作为一个参数传给装饰者的构造函数（装饰器模式的Class Sunwukong），

- 因为装饰类可以装饰不同的对象，例如孙悟空可以保唐僧取经，也可以保唐僧2号。这就带来另外一个区别：使用代理模式，代理和真实对象之间的的关系通常在编译时就已经确定了，而装饰者能够在运行时被构造。

### 桥接模式

桥是用来将河的两岸联系起来的，而设计模式中的桥是用来将两个独立的结构联系起来，而这两个被联系起来的结构可以独立的变化。

我在实现Pegasus的用户自定义compaction策略的时候，曾经用过桥接模式。因为用户自定义策略有两个维度的变量：

1. compaction operation会增多、减少或者修改

2. compaction rule也会增多、减少或修改

对于这种两个独立变化的维度的情况，使用桥接模式可以对其很好的解耦。具体可以参考我之前的[设计文档](https://levy5307.github.io/blog/user-specified-compaction/#%E5%90%8E%E6%9C%9F%E6%89%A9%E5%B1%95)。

### Facade模式

Facade模式是一个很常用的设计模式，其主要用于隐藏系统的复杂性，向客户端提供一个易于访问的接口。

例如，厨房里有面粉、鸡蛋、水和牛奶。根据不同的配比可以做出面包、蛋糕、馒头等等。但是记住这些不同的配比对用户来说太复杂了，所以可以给用户提供一系列的配料表，例如面包配料表，给出了制作面包所需要的原料配比，蛋糕和馒头同理。这样便屏蔽了面包等产品的具体配比等底层复杂信息，简化了客户的操作。这里的配料表就是Facade。

具体代码如下：

```
class Water {
public:
    static std::string get(uint32_t num) {
        return fmt::format("水：{}克", num);
    }
};

class Flour {
public:
    static std::string get(uint32_t num) {
        return fmt::format("面粉：{}克", num);
    }
};

class Egg {
public:
    static std::string get(uint32_t num) {
        return fmt::format("鸡蛋：{}个", num);
    }
};

class Milk {
public:
    static std::string get(uint32_t num) {
        return fmt::format("牛奶：{}克", num);
    }
};

class BurdenSheetFacade {
    std::string bread() {
        std::string burden;
        burden = Flour::get(100) + Egg::get(1) + Water::get(50) + Water::get(20);
        return burden;
    }

    std::string cake() {
        std::string burden;
        burden = Flour::get(100) + Egg::get(3) + Water::get(30) + Milk::get(30);
        return burden;
    }

    std::string maddo() {
        std::string burden;
        burden = Flour::get(100) + Water::get(80);
        return burden;
    }
};
```

通过`BurdenSheetFacade`，可以轻松获取各种不同的配料表，用户操作起来就简单了很多，无需自己去记忆或者查找配料表。这就是Facade的精髓，给客户端提供便利、易操作的接口，屏蔽底层的复杂实现。

### 策略模式

在《设计模式之禅》中，对策略模式的定义如下：

> 定义一组算法，将每个算法都封装起来，并且使他们之间可以互相交换

其类图如下所示：

![](../images/strategy-model.jpg)

其中class Context用于屏蔽具体的ConcreteStrategy。而前面定义中的"算法之间互相交换"是通过将算法抽象化（Strategy）以利用多态来实现的。

在重构Pegasus的load balance代码时，我曾经使用过策略模式。其背景如下：

Pegasus原本有app级别的load balance功能，其认为，只要每个表在集群中负载均衡，那整个集群就是负载均衡的。然而实际线上运维中，我们发现现实情况往往不是这样。因此想要开发集群级别负载均衡功能。最终通过设计选型，我将这部分功能实现如下：

![](../images/images/balancer-strategy-model.jpg)

图中的`greedy_load_balancer`就是class Context，其向上层屏蔽了具体的policy细节，并通过配置来切换使用`app_balance_policy`或者`cluster_balance_policy`

关于load balance重构，可以参考[Pegasus load balance重构](levy5307.github.io/blog/load-balance-refactor/#strategy)

### 命令模式

很多资料中都会讲到，命令模式是一种将命令的请求和命令的执行解耦的模式，并列举出一些例子。很多没有实际工程经验的人，看完其实理解的并不深刻。其实这也是设计模式所面临的问题：学生时期有大把的时间学习、确没有实际经验相结合的精力；工作了以后，很多岗位对代码质量要求也不高，也就很少有人再去深入研究了。

这里希望能够通过逐步分析，帮助了解命令模式最终形态的演进过程。

有这么一个熟悉的场景：小明在操作word，输入一些命令，word文件便会进行不同的操作。例如：输入字符'h'即是普通的输出，而输入Ctrl+C则是复制操作。这里的大概结构如下图：

![](../images/cmd-model-example.jpg)

如图中所示，用户将一些输入数据传递给Computer，Computer根据用户输入，转化成具体的操作系统内部可识别的command信号，并发送给对应的软件（vim或者word）。一个简单的实现如下：

```
class Word {
public:
    Word() = default;

    void handInput(const std::string& input) {
        switch (input) {
            case "J":
		// write "J" to currentFile
		break;
            case "Ctrl+C":
		// do copy operation in currentFile
		break;
        }
    }

private:
    std::string currentFile;
}
```

而小明，在这个例子里也就是客户端，通过直接调用Word类的handleInput来操作文件：

```
class Client {
public:
    Client() = default;

    void process() {
        Word word;
        std::string str = 'J';
        word.handInput(str);

        str = "Ctrl+C";
        word.handInput(str);
    }
};
```

对于当前的简单的功能需求，上述代码是可以满足了。但是它有一个很严重的问题：Word类和具体的命令耦合。当我们需要新添加命令时，需要修改Word实现，可维护性太差了。并且，我们知道很多操作系统是支持自定义按键的，那么当用户修改按键对应的命令时，handleInput的代码也需要修改。

对于耦合，解耦最好的方式就是抽象出来。这里我们可以实现一个虚拟的Command类，其有很多不同的子类，代表着不同的命令，以便将Word类与命令解耦。

```
class File {
public:
    void executeCtrlC() {
        // do copy operation
    }

    void executeJ() {
        std::cout << "J" << std::endl;
    }
};

class Command {
public:
    virtual ~Command() = 0;

    virtual void execute()  = 0
};

class CommandJ : public Command {
public:
    void execute(File *file) {
        file->executeJ();
    }
};

class CommandCtrlC : public Command {
public:
    void execute(File *file) {
        file->executeCtrlC();
    }
};
```

我们再看Word类的实现：

```
class Word {
public:
    Word() {
        // 这里可以采用从配置文件加载的方式来获取input和command的具体对应，以满足用户的自定义按键功能
        // init commands
    }

    void handInput(char input) {
        auto command = getCommand(input);
        command->execute(currentFile);
    }

    void selectFile(File *file) {
        // 这里应该先判断文件是否存在
        currentFile = file;
    }

    File* createFile() {
        files.push_back(new File());
        return files.rend()->get();
    }

private:
    Command* getCommand(char input) const {
        const auto &iter = commands.find(input);
        if (iter != commands.end()) {
            return iter->second->get();
        } else {
            return nullptr;
        }
    }

    File *currentFile;
    std::map<char, std::unique_ptr<Command>> commands;
    std::vector<std::unique_ptr<File>> files;
};
```

客户端代码如下：

```
class Client {
public:
    Client() = default;

    void process() {
        Word word;
        auto file = word.createFile();
        word.selectFile(file);
        word.handInput("J");
        word.handInput("Ctrl+C")
    }
};
```

这里的Word已经不和任何其他实l现耦合了，通过实现Command类，我们将命令的请求者（Word）和命令的执行者（File）隔离开了。这样当我们增加/修改命令时，可以做到不修改class Word实现。

这里可能有人会问，让Client直接操纵File文件不可以吗？如下所示：

```
class Client {
public:
    Client() = default;

    void process() {
        Word word;
        auto file = word.createFile();
        file->executeJ();
        file->executeCtrlC();
    }
};
```

答案当然是不可以了，原因有二：

- Client只关注其自己的输入，对于Word内部执行的是什么命令、Client输入怎么转换成的命令、调用file哪个接口，这些应该是系统内部的实现，应该对客户端进行屏蔽。如果实在不理解，可以想象Word是一个独立的服务，客户端通过网络对其连接，并向Word服务发送输入信息。至于Word服务的内部实现，客户端不应该知道。

![](../images/command-client-network.jpg)

- 用过Word的人都知道，其内部有很多各种各样的功能，比如命令回退、重做、信息统计等。有些功能不应该属于File（因为有很多操作是跨File的），通过直接操纵File文件是很难实现这些复杂功能的。例如，我们要增加一个记录Word中所有文件的命令执行记录统计，具体如下所示：

```
class Word {
public:
    Word() {
        // 这里可以采用从配置文件加载的方式来获取input和command的具体对应，以满足用户的自定义按键功能
        // init commands
    }

    void handInput(char input) {
        auto command = getCommand(input);
        command->execute(currentFile);

	// 统计文件操作命令记录
        commandHistory[currentFile].push_back(command);
    }

    void selectFile(File *file) {
        // 这里应该先判断文件是否存在
        currentFile = file;
    }

    File* createFile() {
        files.push_back(new File());
        return files.rend()->get();
    }

    void statistic() {
        for (const auto &file : histories) {
	    for (const auto &command : file) {
		// print command to a statistic file
	    }
        }
    }

private:
    Command* getCommand(char input) const {
        const auto &iter = commands.find(input);
        if (iter != commands.end()) {
            return iter->second->get();
        } else {
            return nullptr;
        }
    }

    File *currentFile;
    std::map<char, std::unique_ptr<Command>> commands;
    std::vector<std::unique_ptr<File>> files;
    std::map<File*, std::vector<Command*>> commandHistory;
};
```

另外举一个例子，在游戏中的回放功能，其实现也不应该放在游戏角色（Actor）上，因为回放功能不只回放这一个游戏角色的执行命令，而应该是所有角色的命令。

### Adaptor模式

### visitor模式

### 组合模式

### 观察者模式

### 模板方法模式

### 迭代器模式

### 备忘录模式

### 状态模式

### 责任链模式

### 中介模式

### 解释器模式

## 六大设计原则

### 单一职责原则

### 里式替换原则

### 依赖导致原则

### 接口隔离原则

### 迪米特法则

### 开闭原则

## Specific Way In C++ 

### pImpl

### RAII

### Class Trait

## Others

### AOP

### 代码的自解释

### 如何写注释

