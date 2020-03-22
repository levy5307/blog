最近重构rdsn代码，创建了一个logger_proxy类，发现一些单例的线程安全问题，记录如下。

*一、懒汉式*
-------------

最初通过继承singleton的方式来实现的单例，如下：
```
class logger_proxy : public singleton<logger_proxy>
{
}
```
而singleton使用的是静态变量来实现的懒汉式单例：
```
template <typename T>
class singleton
{
public:
    singleton() = default;
 
    // disallow copy and assign
    singleton(const singleton &) = delete;
    singleton &operator=(const singleton &) = delete;
 
    static T &instance()
    {
        static T _instance;
        return _instance;
    }
};
```
在运行dsn\_nfs\_test的时候，当主线程退出时，会子线程会接收到SIGSEGV(signal id = 11)，所以子线程会执行第26行的代码，打印该warn日志。
```
void native_linux_aio_provider::get_event()
{
    struct io_event events[1];
    int ret;
 
    task::set_tls_dsn_context(node(), nullptr);
 
    const char *name = ::dsn::tools::get_service_node_name(node());
    char buffer[128];
    sprintf(buffer, "%s.aio", name);
    task_worker::set_name(buffer);
 
    while (true) {
        if (dsn_unlikely(!_is_running.load(std::memory_order_relaxed))) {
            break;
        }
        ret = io_getevents(_ctx, 1, 1, events, NULL);
        if (ret > 0) // should be 1
        {
            dassert(ret == 1, "io_getevents returns %d", ret);
            struct iocb *io = events[0].obj;
            complete_aio(io, static_cast<int>(events[0].res), static_cast<int>(events[0].res2));
        } else {
            // on error it returns a negated error number (the negative of one of the values listed
            // in ERRORS
            dwarn("io_getevents returns %d, you probably want to try on another machine:-(", ret);
        }
    }
}
```
但是打印该日志，访问到logger_proxy的instance的时候，发生了coredump。也就是说，这时候的instance已经被销毁了
![](../images/sigleton-coredump.png)

***说明这时候的单例并不是真正的线程安全的。***

*二、饿汉式*
-------------

把该单例代码修改成饿汉式：
```
class logger_proxy : public logger
{
public:
    logger_proxy() = default;
    virtual ~logger_proxy(void) = default
 
private:
    static std::unique_ptr<logger_proxy> _instance;
};
 
 
std::unique_ptr<logger_proxy> logger_proxy::_instance =  make_unique<logger_proxy>();
```
再去执行该dsn\_nfs\_test的时候，就不再发生coredump了
![](../images/sigleton-not-coredump.png)

***说明这种饿汉式才是真正的线程安全。***

***TODO：调研为什么懒汉式里的instance会被销毁。***


