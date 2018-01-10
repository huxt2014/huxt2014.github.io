---
layout: post
title:  "gevent基础（二）：基于libev的loop"
date:   2017-08-26 21:00:00 +0800
categories: python
tags: python
---

gevent嫌弃Python实现的ioloop太慢了，所以封装了libev作为自己的ioloop，这个封装是由Cython完成的。这篇文章通过源码深入了解一下gevent中ioloop是如何封装和实现的。

# libev的基本使用

---

gevent封装libev后，提供了调用libev的Python API。Python API的使用方法和libev的C API是类似的，要了解gevent中ioloop的使用方法，还是要从libev开始。以下是libev文档中对io事件提供的使用样例。


{% highlight c %}

#include <ev.h>          /* for libev */
#include <stdio.h>       /* for puts */

ev_io stdin_watcher;     /* a watcher for io */

/* callback for io */
static void
stdin_cb (EV_P_ ev_io *w, int revents)
{
    puts ("stdin ready");
    ev_io_stop (EV_A_ w);  /* unregister the watcher/fd */

    // this causes all nested ev_run's to stop iterating
    ev_break (EV_A_ EVBREAK_ALL);
}

int
main (void)
{
    /* use the default event loop, or you can create a new one */
    struct ev_loop *loop = EV_DEFAULT;

    /* each io watcher will has a callback and a fd */
    ev_io_init (&stdin_watcher, stdin_cb, /*STDIN_FILENO*/ 0, EV_READ);
    /* register the watcher/fd to loop */
    ev_io_start (loop, &stdin_watcher);

    /* start loop and wait for io event */
    ev_run (loop, 0);

    // break was called, so exit
    return 0;
}

{% endhighlight %}

通过文档和样例可以知道，libev将事件以及事件对应的callback函数封装成了watcher。对于io事件，libev定义了ev_io类型的watcher，watcher中记录了fd、事件类型以及callback函数，这些内容通过ev_io_init函数设置。ev_io_start将fd和事件类型注册到ioloop当中，当对应事件触发后就会调用callback函数了。ev_io_stop函数将fd和事件注销掉，ev_break停止ioloop。

除了文件io事件外，libev还提供了其他的事件类型，比如说signal、timer等等，都是封装成了watcher。为简单起见，下文中主要以文件io为例。

# gevent提供的接口

---

了解libev的C API后，再来看看gevent是怎样使用封装之后的ioloop的。

如下BaseServer可以当作是HTTP server，self.socket处于listen状态。为了将socket的io事件加入到ioloop中，gevent调用了loop.io来得到一个watcher对象。调用watcher.start设置callback函数，并且将watcher注册到ioloop当中。对比C API，就能大概猜到背后的逻辑了。

{% highlight python %}
class BaseServer(object):

    def start_accepting(self):
        if self._watcher is None:
            self._watcher = self.loop.io(self.socket.fileno(), 1)
            self._watcher.start(self._do_read)

{% endhighlight %}

# gevent封装io watcher

---

注册一个io事件的关键接口在于loop的io方法和watcher的start方法，gevent的封装方式如下所示：

{% highlight python %}

cdef public class loop [object PyGeventLoopObject, type PyGeventLoop_Type]:

    def io(self, int fd, int events, ref=True, priority=None):
        return io(self, fd, events, ref, priority)


cdef public class io(watcher) [object PyGeventIOObject, type PyGeventIO_Type]:
    
    cdef public loop loop
    cdef object _callback           # Python callable object
    cdef public tuple args
    cdef readonly int _flags
    cdef libev.ev_io _watcher

    def __init__(self, loop loop, int fd, int events, ref=True, priority=None):
        ...
        # 初始化，设置了fd、事件以及callback
        libev.ev_io_init(&self._watcher, <void *>gevent_callback_io, fd,
                         events)
        self.loop = loop
        if ref:
            self._flags = 0
        else:
            self._flags = 4
        if priority is not None:
            libev.ev_set_priority(&self._watcher, priority)

    def start(self, object callback, *args, pass_events=False):
        ...
        # 之前注册的callback会调用这个
        self.callback = callback
        ...
        # 注册到loop当中
        libev.ev_io_start(self.loop._ptr, &self._watcher)
        ...

# 这个就是上面注册的callback了，展开宏之后得到
static void gevent_callback_io(struct ev_loop *_loop, void *c_watcher,
                               int revents) {

    # 用c_watcher的地址可以找到对应PyGeventIOObject的地址，按照地址偏移量来确定
    struct PyGeventIOObject* watcher = GET_OBJECT(PyGeventIOObject, c_watcher,
                                                  _watcher);
    # 调用 Python callable object
    gevent_callback(watcher->loop, watcher->_callback, watcher->args,
                    (PyObject*)watcher, c_watcher, revents);
}

{% endhighlight %}

# gevent封装ev_preapre

---

在一个loop中，通常都会有一系列callback在每次loop之前／之后执行，这些callback可能是一次性的，也可能是周期性的。gevent自己的loop当然也有这个功能，通过loop.run_callback向外提供了接口。对于run_callback的实现，gevent利用了libev的一个用于扩展的watcher：ev_prepare，如下所示：

{% highlight python %}

cdef public class loop [object PyGeventLoopObject, type PyGeventLoop_Type]:
    cdef libev.ev_loop* _ptr
    cdef public object error_handler
    cdef libev.ev_prepare _prepare
    cdef public list _callbacks
    cdef libev.ev_timer _timer0

    def __init__(self, object flags=None, object default=None, size_t ptr=0):
        cdef unsigned int c_flags
        cdef object old_handler = None
        # 初始化prepare watcher，在每次loop之前都会调用gevent_run_callbacks
        libev.ev_prepare_init(&self._prepare, <void*>gevent_run_callbacks)
        libev.ev_timer_init(&self._timer0, <void*>gevent_noop, 0.0, 0.0)

        if default:
            self._ptr = libev.gevent_ev_default_loop(c_flags)
        else:
            self._ptr = libev.ev_loop_new(c_flags)

        # 注册到loop当中
        libev.ev_prepare_start(self._ptr, &self._prepare)
        libev.ev_unref(self._ptr)
        self._callbacks = []

    def run_callback(self, func, *args):

        if not self._ptr:
            raise ValueError('operation on destroyed loop')
        cdef callback cb = callback(func, args)
        self._callbacks.append(cb)
        libev.ev_ref(self._ptr)
        return cb

    cdef _run_callbacks(self):
        cdef callback cb
        cdef object callbacks
        cdef int count = 1000
        libev.ev_timer_stop(self._ptr, &self._timer0)
        while self._callbacks and count > 0:
            callbacks = self._callbacks
            self._callbacks = []
            for cb in callbacks:
                libev.ev_unref(self._ptr)
                gevent_call(self, cb)
                count -= 1
        if self._callbacks:
            libev.ev_timer_start(self._ptr, &self._timer0)


static void gevent_run_callbacks(struct ev_loop *_loop, void *watcher,
                                 int revents) {
    struct PyGeventLoopObject* loop;
    PyObject *result;
    GIL_DECLARE;
    GIL_ENSURE;
    # 用watcher的地址可以找到PyGeventLoopObject的地址，通过地址偏移来确认
    loop = GET_OBJECT(PyGeventLoopObject, watcher, _prepare);
    Py_INCREF(loop);
    gevent_check_signals(loop);
    # 貌似就是loop定义中的 cdef _run_callbacks
    result = ((_GEVENTLOOP *)loop->__pyx_vtab)->_run_callbacks(loop);
    if (result) {
        Py_DECREF(result);
    }
    else {
        PyErr_Print();
        PyErr_Clear();
    }
    Py_DECREF(loop);
    GIL_RELEASE;

{% endhighlight %}


# 小结

---

虽然说Cython让写C的扩展方便了许多，但是碰到gevent这样大量使用宏来减少代码量，看起来也是比较费力。

理解了libev的使用方法和gevent的封装实现之后，只要理解怎样往loop中注册事件就可以了，即主要通过以下两种方式：
1. 通过loop对象的接口得到watcher实例，这样注册的callback会在loop当中去调用。

2. 通过run_callback注册事件，这样注册的callback会在每次loop循环之前调用。

(the end)
