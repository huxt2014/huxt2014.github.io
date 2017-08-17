---
layout: post
title:  "Python基础：各类并发中的future对象"
date:   2017-01-21 14:48:08 +0800
categories: python
tags: python
---

虽然Python有GIL的限制，但并发并不是问题。Python的标准库提供了[threading包][threading-doc]以及[multiprocessing包][multiprocessing]来支持线程与进程的并发，在Python 3中更是提供了[concurrent 包][concurrent-doc]这一更高层的封装。此外，Python 3还通过新增的await、async def关键字，以及[asyncio包][asyncio-doc]提供了native coroutine的支持。要深入了解Python的并发，这些标准库中的包是不错的学习材料。

最近在看concurrent、asyncio这两个包的源码，发现future对象在线程、进程、coroutine中都出现了，并且起到了比较重要的作用。仔细了解和对比过后，用这篇文章来简单记录一下对于future的理解。

总体来讲，future对象起到的作用有三个：占位(placeholder)，回调（callback）以及同步/异步等待。

# future in ThreadPoolExecutor

先来看一个简单的例子，如下所示。

{% highlight python %}
import time
from datetime import datetime
from concurrent.futures import ThreadPoolExecutor

def wait():
    time.sleep(5)
    return 'hello'

executor = ThreadPoolExecutor(max_workers=2)

print(datetime.now(), 'submit')
a = executor.submit(wait)
print(datetime.now(), 'submit return')

print(datetime.now(), 'get result %s' % a.result())
print(datetime.now())

# output:
#=> 2017-01-21 12:10:16.989216 submit
#=> 2017-01-21 12:10:16.990252 submit return
#=> 2017-01-21 12:10:16.990340 get result hello
#=> 2017-01-21 12:10:21.996463

{% endhighlight %}

第一次使用ThreadPoolExecutor的时候，对以上执行结果感觉颇为迷惑。submit表示要执行wait函数，wait函数中sleep了5秒，为什么submit后当前线程立即得到了结果？第三条print的时间为什么还是没有显示sleep 5秒这个逻辑？为什么最后一条print才显示出函数执行完了？

实际上，submit一个函数后，对应函数会被包装在一个_WorkItem对象中，与之包装在一起的是一个future对象。不管函数是否执行完了，future对象会被立即返回。这个future对象可以这么理解：未来某个时候future对象中会存在函数的返回结果。_WorkItem会被放到队列中，由ThreadPoolExecutor中的thread worker处理，执行完后，执行单位会把函数的执行结果写到future对象中。在这里，future起到了placeholder的作用。

得到future对象后，当前线程可以执行其他的操作，当需要时可以调用future.result()获得结果。如果_WorkItem还没被处理完，future.result()会阻塞当前线程。在这里，futre起到了同步等待的作用。至于同步等待是如何实现的，以及future是如何实现回调的，那就要从源码中找找答案了。

{% highlight python %}
# concurrent/futures._base.py

class Future(object):
    """Represents the result of an asynchronous computation."""

    def __init__(self):
        """Initializes the future. Should not be called by clients."""
        self._condition = threading.Condition()
        self._state = PENDING
        self._result = None
        self._exception = None
        self._waiters = []
        self._done_callbacks = []
        
    def result(self, timeout=None):
        with self._condition:
            if self._state in [CANCELLED, CANCELLED_AND_NOTIFIED]:
                raise CancelledError()
            elif self._state == FINISHED:
                return self.__get_result()

            self._condition.wait(timeout)     # 阻塞

            if self._state in [CANCELLED, CANCELLED_AND_NOTIFIED]:
                raise CancelledError()
            elif self._state == FINISHED:
                return self.__get_result()
            else:
                raise TimeoutError()
                
    def set_result(self, result):

        with self._condition:
            self._result = result
            self._state = FINISHED
            for waiter in self._waiters:
                waiter.add_result(self)
            self._condition.notify_all()      # 唤醒阻塞
        self._invoke_callbacks()              # 执行回调函数
   
{% endhighlight %}

以上列出了Future类以及三个主要的函数。从初始化函数中可以看到，Future对象除了有_result，里面还有_condition和_done_callback。其中，_condition就是同步等待的关键了。

在调用result函数的时候，如果future已经完成了，那么会直接返回结果；如果future还没有完成，那么利用调用_condition.wait阻塞当前线程。ThreadPoolExecutor中的thread worker执行完对应函数后，会调用set_result。在该函数中，除了调用_condition.notify_all唤醒其他被阻塞的线程，还调用了所有的_done_callback。

以上就是future如何实现placeholder，callback以及同步等待的。

# future in ProcessPoolExecutor

上面介绍完了future在线程并发中的作用，接下来看看ProcessPoolExecutor封装的进程并发。

从源码中可以看到，ProcessPoolExecutor用的future对象和ThreadPoolExecutor中用到的future对象属于同一个类，提供了几乎一样的功能。初看之下感觉很不合理啊，因为Future类中同步用的是threading.Condition，理应不能用到进程间同步的啊。这其中的原因，就要从ProcessPoolExecutor的设计思想中寻找了。

ProcessPoolExecutor对象存在于当前进程中，其负责管理当前进程中的一个后台线程，该线程负责与Process worker通信（发送执行任务以及获取执行结果）。获得执行结果后，操作future对象的是该后台线程，因此future对象可以用到threading.Condition。 

具体来讲， 当前线程调用submit后，需要执行的函数以及future对象一方面会被包装到_WorkItem中，_WorkItem会被保存在当前进程中；另一方面需要执行的函数、_WorkItem的索引会被包装成 _CallItem对象，然后发送到process worker。process worker处理完_CallItem后，会返回结果以及对应_WorkItem对象的索引，后台线程根据索引找到_WorkItem对象并操作future对象。

理解以上原理后，很容易就能在源码中找到对应的实现了。

# future in asyncio

在asyncio模块中，依然有Future类的存在，这个类的实现独立于concurrent包中的Future类，但是有类似的功能：placeholder、callback以及等待。不同的是，这个Future类用于coroutine中，所谓的等待是异步等待。

{% highlight python %}
# asyncio/futures.py

class Future:
    _result = None
    
    def __init__(self, *, loop=None):

        if loop is None:
            self._loop = events.get_event_loop()
        else:
            self._loop = loop
        self._callbacks = []
        if self._loop.get_debug():
            self._source_traceback = traceback.extract_stack(sys._getframe(1))
    
    def __iter__(self):
        if not self.done():
            print(datetime.now())
            self._blocking = True
            yield self  # This tells Task to wait for completion.
        print(datetime.now())
        assert self.done(), "yield from wasn't used with future"
        return self.result()  # May raise too.

    if compat.PY35:
        __await__ = __iter__ # make compatible with 'await' expression

{% endhighlight %}

以上摘录的部分源码，_result与_callbacks为placeholder和callback提供了支持，而coroutine的异步等待，则由__iter__ 以及__awat__提供支持。具体的实现原理将在介绍native coroutine的文章中详细说明。

[threading-doc]: https://docs.python.org/3/library/threading.html
[multiprocessing]: https://docs.python.org/3.3/library/multiprocessing.html
[concurrent-doc]: https://docs.python.org/3/library/concurrent.html
[asyncio-doc]: https://docs.python.org/3.6/library/asyncio.html
