---
layout: post
title:  "gevent基础（三）：像操作线程一样操作协程"
date:   2017-08-27 21:00:00 +0800
categories: python
tags: python
---

基于异步IO来实现并发的框架都会有一个loop来监听各种事件，当相关事件发生的时候，loop会作出响应继而调用对应的callback函数。不同的框架中，虽然内部的实现原理都是一样的，但是暴露给外部的API却各有各的特点。例如，pika（RabbitMQ的Python客户端）需要明确地设置callback函数，Python 3的asyncio利用await和yeild from提供了更高级的接口。

在gevent中，callback的模式被隐藏地更深，以至于很多API使用起来和同步编程风格很像，所以被称为隐式（implicit）的异步框架。上一篇文章[gevent基础（二）：基于libev的loop]({{site.baseurl}}{% post_url 2017-08-26-gevent-loop %})中已经介绍了gevent的loop是怎样实现的，在这基础之上，这篇文章来了解一下gevent是怎样实现像线程一样的接口的。


# greenlet之间的依赖关系

---

先来看看下面一个简单的例子：main函数中用gevent.spawn启动了一个协程g，然后调用g.join等待协程执行完毕，最后main函数执行完毕。看起来有没有像线程的使用方式？有sleep，有join，执行结果也和预期的一样，但是没有loop和callback的任何痕迹！

{% highlight python %}

import gevent

def func():
    print("hello")
    gevent.sleep(2)

def main():
    print("main begin")
    g = gevent.spawn(func)
    g.join()
    print("main end")

if __name__ == "__main__":
    main()

#输出：
#main begin
#hello
#main end

{% endhighlight %}

接下来通过查看源码来看看以上效果是如何实现的吧。gevent.spwan就是Greenlet类的spwan方法，而Greenlet继承自greenlet，所有在应用程序中创建的协程都是Greenlet的实例，所以可以认为Greenlet就是协程的载体（除非有明确说明，下文中不区分greenlet、Greenlet和协程）。更多的说明见注释。

{% highlight python %}

class Greenlet(greenlet):
    # 继承greenlet

    def __init__(self, run=None, *args, **kwargs):
        # 所有的协程都有一个共同的parent
        greenlet.__init__(self, None, get_hub())

        if run is not None:
            self._run = run
        ...

    @classmethod
    def spawn(cls, *args, **kwargs):
        g = cls(*args, **kwargs)
        g.start()
        return g

    def start(self):
        # 将switch放到loop的callback中，loop调用switch后就会切换到该协程，
        # 然后会运行run方法
        if self._start_event is None:
            self._start_event = self.parent.loop.run_callback(self.switch)

    def run(self):
        # 重载了run方法，调用 self._run
        ...


def get_hub(*args, **kwargs):

    # 每个线程中只有一个hub
    hub = _threadlocal.hub
    if hub is None:
        hubtype = get_hub_class()
        hub = _threadlocal.hub = hubtype(*args, **kwargs)
    return hub


class Hub(RawGreenlet):
    # 继承greenlet

    def __init__(self, loop=None, default=None):
        # initial self.loop
        ...

    def run():
        # 重载了run方法，调用switch切换到hub后就会启动loop
        ...


{% endhighlight %}

乍看之下没那么复杂，结构很简单，但是仔细思考一下之后会发现，程序中所有greenlet之间的关系是值得特别注意的。就某一个线程而言，其中有一个root greenlet，然后有一个hub greenelt，之后是各类Greenlet，它们的关系及承载的逻辑如下所示。这种逻辑和普通的异步io框架有一个最大的区别，那就gevent中main函数和loop属于不同的greenlet，而普通的异步io框架loop是在mian函数当中的，理解这个对于理解应用程序中各种逻辑的切换有很大帮助。

{% highlight python %}

  root greenlet       hub greenlet            other greenlets
   +--------+          +--------+              +-----------+
   |  main  |  <-----> |  loop  |  <----+--->  | coroutine |
   +--------+          +--------+       |      +-----------+
                                        |      +-----------+
                                        +--->  | coroutine |
                                        |      +-----------+
                                        +--->  ...
{% endhighlight %}


# join

---

join在线程当中用于阻塞当前线程的运行，等待目标线程执行完毕，在gevent中的效果也是这样，只不过是通过异步io来实现的。从下面源码可以看到，对于“挂起逻辑A，等逻辑B执行完成后再执行A”这类操作的实现，gevent和其他异步io框架是类似的。对于上述的例子而言，就是将main挂载到g的_links上，然后切换到hub上，hub中loop会调度g的运行，g运行完成之后会将调用_links上挂载的东西，也就是返回到main当中。

对于Timeout的实例，其实现的原理就是通过loop.timer启动一个定时任务，这个任务就是抛出异常。

{% highlight python %}
class Greenlet(greenlet):

    def join(self, timeout=None):
        ...
        # 当前的greenlet，可能是main，也可能是其他协程
        switch = getcurrent().switch
        self.rawlink(switch)
        try:
            t = Timeout._start_new_or_dummy(timeout)
            try:
                # 所有Greenlet的parent都是hub
                result = self.parent.switch()
                if result is not self:
                    raise InvalidSwitchError(
                             'Invalid switch into Greenlet.join(): %r'
                             % (result, ))
            finally:
                t.cancel()
        except Timeout as ex:
            self.unlink(switch)
            if ex is not t:
                raise
        except:
            self.unlink(switch)
            raise

    def rawlink(self, callback):
        ...
        self._links.append(callback)
        if self.ready() and self._links and not self._notifier:
            self._notifier = self.parent.loop.run_callback(self._notify_links)

    @Lazy
    def _links(self):
        return deque()

    def run(self):
        result = self._run(*self.args, **self.kwargs)
        self._report_result(result)

    def _report_result(self, result):
        self._exc_info = (None, None, None)
        self.value = result
        if self._has_links() and not self._notifier:
            # 放到loop当中
            self._notifier = self.parent.loop.run_callback(self._notify_links)

    def _notify_links(self):
        while self._links:
            link = self._links.popleft()
            try:
                link(self)
            except: # pylint:disable=bare-except
                self.parent.handle_error((link, self), *sys.exc_info())

class Timeout(BaseException):

    def __init__(self, seconds=None, exception=None, ref=True, priority=-1, 
                 _use_timer=True):
        BaseException.__init__(self)
        self.seconds = seconds
        self.exception = exception
        if seconds is None or not _use_timer:
            self.timer = _FakeTimer
        else:
            self.timer = get_hub().loop.timer(seconds or 0.0, ref=ref,
                                              priority=priority)

    def start(self):
        """Schedule the timeout."""
        assert not self.pending
        if self.seconds is None:  # "fake" timeout (never expires)
            return

        if (self.exception is None
           or self.exception is False
           or isinstance(self.exception, string_types)):
            # timeout that raises self
            self.timer.start(getcurrent().throw, self)
        else:  # regular timeout with user-provided exception
            self.timer.start(getcurrent().throw, self.exception)

    @classmethod
    def start_new(cls, timeout=None, exception=None, ref=True):
        ...
        timeout = cls(timeout, exception, ref=ref)
        timeout.start()
        return timeout

    @staticmethod
    def _start_new_or_dummy(timeout, exception=None):

        if timeout is None:
            return _FakeTimer
        return Timeout.start_new(timeout, exception)

{% endhighlight %}

# sleep

---

对于sleep的实现方法，理解下面的源码后也就一目了然了。其中值得注意的是，Gevent通过Waiter提供了一个相对通用的接口。

{% highlight python %}

def sleep(seconds=0, ref=True):
    hub = get_hub()
    loop = hub.loop
    if seconds <= 0:
        waiter = Waiter()
        # 相当于 loop.run_callback(getcurrent.switch)
        loop.run_callback(waiter.switch)
        # 相当于hub.switch()
        waiter.get()
    else:
        hub.wait(loop.timer(seconds, ref=ref))


class Waiter(object):

    def switch(self, value=None):
        greenlet = self.greenlet
        if greenlet is None:
            self.value = value
            self._exception = None
        else:
            assert getcurrent() is self.hub
            switch = greenlet.switch
            try:
                switch(value)
            except:
                self.hub.handle_error(switch, *sys.exc_info())

    def get(self):
        if self._exception is not _NONE:
            if self._exception is None:
                return self.value
            else:
                getcurrent().throw(*self._exception)
        else:
            if self.greenlet is not None:
                raise ConcurrentObjectUseError()
            self.greenlet = getcurrent()
            try:
                return self.hub.switch()
            finally:
                self.greenlet = None


class Hub(RawGreenlet):

    def wait(self, watcher):
        waiter = Waiter()
        unique = object()
        # 相当于watcher的事件发生之后执行getcurrent().switch
        watcher.start(waiter.switch, unique)
        try:
            # 相当于hub.switch()
            result = waiter.get()
            if result is not unique:
                raise InvalidSwitchError()
        finally:
            watcher.stop()

{% endhighlight %}


# 小结

---

在基于Gevent的应用程序当中，有三类greenlet：root greenlet（执行main），hub greenlet（执行loop），其他Greenlet（协程）。其中，root greenlet与Greenlet的实例有并列的关系，由hub greenlet进行调度。

Gevent封装了比较高级的接口，使得协程的操作比较像线程的操作。在这种封装之下，root greenlet与Greenlet之间的切换有了统一的模式：当某个greenlet需要切换到另外一个greenlet时，需要先切换到hub，由hub来调度另外一个greenlet。


(the end)
