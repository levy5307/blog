kudu的CMakeList中添加了-DNDEBUG，如下所示：
```
set(CXX_FLAGS_RELEASE "-O3 -g -DNDEBUG -fno-omit-frame-pointer")
```
当设置了NDEBUG时，assert的定义则为：
```
#ifdef NDEBUG
#define assert(condition) ((void)0)
#else
#define assert(condition)  /* implementation defined */
#endif
```

# Reference 
https://murphypei.github.io/blog/2020/01/assert-debug-release.html
