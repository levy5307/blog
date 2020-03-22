Suppose you have a class X that has a static Fred object:
```c++
// File X.h
class X {
public:
  // ...
private:
  static Fred x_;
};
```
Naturally this static member is initialized separately:
```c++
// File X.cpp
#include "X.h"
Fred X::x_;
```

Naturally also the Fred object will be used in one or more of X’s methods:
```c++
void X::someMethod()
{
  x_.goBowling();
}
```

But now the “disaster scenario” is if someone somewhere somehow calls this method before the Fred object gets constructed. For example, if someone else creates a static X object and invokes its someMethod() method during static initialization, then you’re at the mercy of the compiler as to whether the compiler will construct X::x_ before or after the someMethod() is called. (Note that the ANSI/ISO C++ committee is working on this problem, but compilers aren’t yet generally available that handle these changes; watch this space for an update in the future.)
In any event, it’s always portable and safe to change the X::x_ static data member into a static member function:
```c++
// File X.h
class X {
public:
  // ...
private:
  static Fred& x();
};
```
Naturally this static member is initialized separately:
```c++
// File X.cpp
#include "X.h"
Fred& X::x()
{
  static Fred* ans = new Fred();
  return *ans;
}
```
Then you simply change any usages of x_ to x():
```c++
void X::someMethod()
{
  x().goBowling();
}

```
If you’re super performance sensitive and you’re concerned about the overhead of an extra function call on each invocation of X::someMethod() you can set up a static Fred& instead. As you recall, static local are only initialized once (the first time control flows over their declaration), so this will call X::x() only once: the first time X::someMethod() is called:
```c++
void X::someMethod()
{
  static Fred& x = X::x();
  x.goBowling();
}
```
