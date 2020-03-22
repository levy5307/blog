最近工作中发现了一个关于智能指针的=运算符重载的经典代码实现，记录一下：

```
  scoped_refptr& operator=(std::nullptr_t) {
    reset();
    return *this;
  }
  // Sets managed object to null and releases reference to the previous managed
  // object, if it existed.
  void reset() { scoped_refptr().swap(*this); }

  scoped_refptr& operator=(T* p) { return *this = scoped_refptr(p); }
  // Unified assignment operator.
  scoped_refptr& operator=(scoped_refptr r) noexcept {
    swap(r);
    return *this;
  }
  void swap(scoped_refptr& r) noexcept { std::swap(ptr_, r.ptr_); }
```

可以只看operator=(std::nullptr_t), 由于将this赋值给null，所以this需要释放。此时operator=这个函数不好实现，因为如果在函数退出前释放的话，释放后的任何操作都相当于操作了已释放对象(可参考附录1我在pr中的描述)。

这里的google的实现是：将this和一个临时变量进行swap, 该临时对象的ptr_为null，交换后临时对象的ptr_变成了this->ptr_，this->ptr_变成了null，这是前提。然后经典的来了，当reset函数退出时，

临时变量退出了其作用区域，会被释放，根据指针的特性，其对应的其ptr_指向的空间也会释放，由于此时临时对象的ptr_是this->ptr_，所以reset退出时，实际释放的是this->ptr_。

下面的几个函数原理是一样的，这里就不解释了

具体代码文件：https://chromium.googlesource.com/chromium/src/+/master/base/memory/scoped_refptr.h

附录1：https://github.com/XiaoMi/rdsn/pull/361
