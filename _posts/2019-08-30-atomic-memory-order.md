虽然是六种类型，但是理解了四种同步的情形基本就差不多了。

对于单线程来说，编译器会在不影响程序执行结果的前提下对原子语句顺序进行重排，同时有些弱序cpu(spark是弱序，但x86是强序)也会对指令执行顺序进行重排。但是这种重拍对于多线程来说可能就会造成问题。

1. Sequential consistency: 即对于所有的环境变量，代码在线程中运行的顺序与程序员看到的代码顺序一致。对于弱序cpu需要添加内存栅栏（mem fence）

2. Relaxed ordering: 不保证顺序，性能最高。有可能会由于编译器或者cpu的重排导致程序运行出问题。

3. Release -- acquire: Release保证在其之前的(所有)语句一定会先与该Release语句执行, Acquire语句保证在其之后的（所有）语句一定会后语该语句执行。Release用于store，Acquire用于load。 

4. Release -- consume: 如果只想同步一个 x 的读写操作，结果把 release 之前的写操作都顺带同步了？如果想避免这个额外开销怎么办？用 release -- consume。同步还是一样的同步，这回副作用弱了点：memory_order_cosume只保证原子操作发生在与其（memory_order_consume指定的）有关的原子操作之前，对于无关的操作则无法保证顺序。例如：

atomic<string*> ptr;
atomic<int> data;

producer:

	string *p = new string("hello");

	data.store(42, memory_order_relaxed);

	ptr.store(p, memory_order_release);

consumer:

	string *p2;

	while(!(p2 = ptr.load(memory_order_consume));

	assert(*p2 == "hello");

	assert((data.load(memory_order_relaxed) == 42);

这可以保证assert(*p2 == "hello")运行在while(!(p2 = ptr.load(memory_order_consume))之后，但是无法保证assert((data.load(memory_order_relaxed) == 42)运行在其之后

5. atomic_flag相对于atomic<T>的优势是atomic_flag一定无锁，但是atomic<T>确不一定，如果有些平台不为T类型提供lock-free的atomic操作，那么atomic<T>则会用mutex来实现，这样肯定就不是lock-free了，
具体是否lock-free可以通过其成员函数is_lock_free()来判断。引用stackflow：

The standard does not specify if atomic objects are lock-free. On a platform that doesn't provide lock-free atomic operations for a type T, atomic<T> objects may be implemented using a mutex, 
which wouldn't be lock-free. In that case, any containers using these objects in their implementation would not be lock-free either.

The standard does provide a way to check if an atomic<T> variable is lock-free: you can use var.is_lock_free() or atomic_is_lock_free(&var). 
These functions are guaranteed to always return the same value for the same type T on a given program execution. For basic types such as int, There are also macros provided (e.g. ATOMIC_INT_LOCK_FREE) which specify if lock-free atomic access to that type is available


具体参考《深入理解C++11 新特性解析与应用》P196-P213 原子类型与原子操作
