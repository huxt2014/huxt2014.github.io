---
layout: post
title:  "asyncio与协程并发"
date:   2017-02-03 10:00:00 +0800
categories: python
---

asyncio是Python 3中加入的标准库，在这个标准库中加入了ioloop以及对native coroutine的支持。对于ioloop，基本上所有的异步框架都会有，例如gevent、tornado、pika，它们实现的原理都是一样的，只是经过不同框架的封装之后使用接口略有不同，看源码都是很容易理解的，所以就不多说了。但是对于native coroutine，由于之前没深入了解过，所以就多用点文字总结一下看完部分源码后的理解。

有了前面两篇文章对future对象和async def、await关键字进行的铺垫，通过看asyncio的源码来了解navtive coroutine的并发会顺畅很多，这篇文章详细说说sleep函数和异步IO的实现。

# sleep 函数

对于coroutine，以及sleep的使用，就从官方示例开始说起吧，示例如下所示。

{% highlight python %}

import asyncio

async def compute(x, y):
    print("Compute %s + %s ..." % (x, y))
    await asyncio.sleep(1.0)
    return x + y

async def print_sum(x, y):
    result = await compute(x, y)
    print("%s + %s = %s" % (x, y, result))

loop = asyncio.get_event_loop()
loop.run_until_complete(print_sum(1, 2))
loop.close()

{% endhighlight %}

根据官方文档描述，调用loop.run_until_complete并传入corontine后，coroutine会包装到task对象中，让task对象负责coroutine的调度（这和concurrent.futures包中的_WorkItem有点像吧，都需要包装一下）。扫一遍源码很快能发现run_until_complete都干了啥：把coroutine包装到task对象中，并将task对象的_step方法放到了ioloop中让其立即执行。看来_step方法就是关键所在了，把源码择出来如下。

{% highlight python %}
# asyncio/tasks.py

class Task(futures.Future):
    # 省略...
    
    def _step(self, exc=None):
        # 省略...
        coro = self._coro
        self._fut_waiter = None

        self.__class__._current_tasks[self._loop] = self
        # Call either coro.throw(exc) or coro.send(None).
        try:
            if exc is None:
                # We use the `send` method directly, because coroutines
                # don't have `__iter__` and `__next__` methods.
                result = coro.send(None)
            else:
                result = coro.throw(exc)
        except StopIteration as exc:
            self.set_result(exc.value)
        except futures.CancelledError as exc:
            super().cancel()  # I.e., Future.cancel(self).
        except Exception as exc:
            self.set_exception(exc)
        except BaseException as exc:
            self.set_exception(exc)
            raise
        else:
            if isinstance(result, futures.Future):
                # Yielded Future must come from Future.__iter__().
                if result._loop is not self._loop:
                    self._loop.call_soon(
                        self._step,
                        RuntimeError(
                            'Task {!r} got Future {!r} attached to a '
                            'different loop'.format(self, result)))
                elif result._blocking:
                    result._blocking = False
                    result.add_done_callback(self._wakeup)
                    self._fut_waiter = result
                    if self._must_cancel:
                        if self._fut_waiter.cancel():
                            self._must_cancel = False
                else:
                    self._loop.call_soon(
                        self._step,
                        RuntimeError(
                            'yield was used instead of yield from '
                            'in task {!r} with {!r}'.format(self, result)))
            elif result is None:
                # Bare yield relinquishes control for one event loop iteration.
                self._loop.call_soon(self._step)
            elif inspect.isgenerator(result):
                # Yielding a generator is just wrong.
                self._loop.call_soon(
                    self._step,
                    RuntimeError(
                        'yield was used instead of yield from for '
                        'generator in task {!r} with {}'.format(
                            self, result)))
            else:
                # Yielding something else is an error.
                self._loop.call_soon(
                    self._step,
                    RuntimeError(
                        'Task got bad yield: {!r}'.format(result)))
        finally:
            self.__class__._current_tasks.pop(self._loop)
            self = None  # Needed to break cycles when an exception occurs.

    def _wakeup(self, future):
        try:
            future.result()
        except Exception as exc:
            # This may also be a cancellation.
            self._step(exc)
        else:
            self._step()
        self = None  # Needed to break cycles when an exception occurs.

    # 省略...
{% endhighlight %}

很多if else有木有，很多except有木有，看起来略感蛋疼。既然这样，那就抓住主线看吧。_step方法被调用后，首先会调用coroutine的send方法。前面的文章提到过，不管coroutine里面嵌套了多少层coroutine，send方法都是和最里面的yield交互。官方示例中，print_sum调用了compute，compute调用了sleep，那么yield势必就在sleep里面了。

{% highlight python %}
# asyncio/tasks.py

@coroutine
def sleep(delay, result=None, *, loop=None):
    """Coroutine that completes after a given time (in seconds)."""
    if delay == 0:
        yield
        return result

    if loop is None:
        loop = events.get_event_loop()
    future = loop.create_future()
    h = future._loop.call_later(delay,
                                futures._set_result_unless_cancelled,
                                future, result)
    try:
        return (yield from future)
    finally:
        h.cancel()

{% endhighlight %}

从sleep的定义可以看到，如果delay=0，那么会yield一个None。所以_step中第一次执行send后，得到的是None。得到None后_step被挂到ioloop中，然后返回。这个时候，ioloop就可以去干其他事了。

当ioloop再次调度的时候，上述挂起的_step会再次执行。当_step再次执行send的时候，sleep会抛出StopIteration+result，这个异常会被computer中的await asyncio.sleep捕获到，然后这个表达式的值就是result了。await exression获得值后，就会像普通函数那样执行之后的代码，直到再次遇到yield或者coroutine结束。

从这里可以看出，如果delay=0，就相当于主动让出cpu，等待ioloop的下一次调度。

如果delay大于0，sleep中生成了一个future对象，通过_loop.call_later方法在delay时间后调用这个future对象的set_result方法，然后yield from future。future遵循PEP 492中的规范，定义了__await__方法，该方法返回一个iterator，所以可以被await。下面来看看future中__await__的实现。

{% highlight python %}
# asyncio/futures.py

class Future:
    # 省略...
    def __iter__(self):
        if not self.done():
            self._blocking = True
            yield self  # This tells Task to wait for completion.
        
        assert self.done(), "yield from wasn't used with future"
        return self.result()  # May raise too.

    if compat.PY35:
        __await__ = __iter__ # make compatible with 'await' expression
        
    # 省略...
 {% endhighlight %}
 
_step中第一次send时，sleep中的future对象是没有完成的。所以，这个时候send与future对象中的yield交互，然后返回这个future对象。_step拿到future对象后，将_wakeup放到future的done_callback中，然后结束，ioloop可以执行其他逻辑了。

虽然_step返回了，ioloop去干其他事了，但以上故事并没有结束。delay时间后，future对象的set_result被调用，然后future对象的done_callback被调用，然后_wakeup被调用，最后又执行了_step。这次执行_step的时候，通过send再次进入到future.__iter__中的逻辑。这个时候future已经完成了，所以会执行return。

之后的执行逻辑就显而易见了，通过StopIteration依次返回结果。最后的效果就是compute中的await asyncio.sleep(1.0)等待了1s的时间，在这1s中，ioloop可以去干其他事情，而不是被block掉。

# socket 异步IO

理解了上述过程后，对于socket的异步IO也会很好理解了，这里来看一下BaseSelectorEventLoop.sock_recv的实现。
{% highlight python %}
# asyncio/selector_events.py

class BaseSelectorEventLoop(base_events.BaseEventLoop):
    def sock_recv(self, sock, n):
        """Receive data from the socket.

        The return value is a bytes object representing the data received.
        The maximum amount of data to be received at once is specified by
        nbytes.

        This method is a coroutine.
        """
        if self._debug and sock.gettimeout() != 0:
            raise ValueError("the socket must be non-blocking")
        fut = self.create_future()
        self._sock_recv(fut, False, sock, n)
        return fut

    def _sock_recv(self, fut, registered, sock, n):
        # _sock_recv() can add itself as an I/O callback if the operation can't
        # be done immediately. Don't use it directly, call sock_recv().
        fd = sock.fileno()
        if registered:
            # Remove the callback early.  It should be rare that the
            # selector says the fd is ready but the call still returns
            # EAGAIN, and I am willing to take a hit in that case in
            # order to simplify the common case.
            self.remove_reader(fd)
        if fut.cancelled():
            return
        try:
            data = sock.recv(n)
        except (BlockingIOError, InterruptedError):
            self.add_reader(fd, self._sock_recv, fut, True, sock, n)
        except Exception as exc:
            fut.set_exception(exc)
        else:
            fut.set_result(data)

{% endhighlight %}

注释表示这个方法是coroutine，其实我觉得称之为awaitable更准确。这是一个普通的函数，之所以可以用于await，是因为它返回一个future对象，而future可以用于await。下面来看看这个方法的执行逻辑。

首先，sock应该是处于非阻塞模式的。如果sock可以读到指定长度的数据，那么数据会写到future中，然后该future被返回。从future的实现中可以看到，await一个已经有结果的future会立即抛出StopIteration+result，然后await作为一个expression就有值了。这个时候是不会被挂起的。

如果sock没办法读到指定长度的数据，那么会抛出BlockingIOError，这个时候sock会放到ioloop中监听读事件。虽然future中没有结果，但是依然会被立即返回。await一个未完成的future，执行逻辑和sleep是一样的。

每次sock可读后，_sock_recv都会被再次被执行，直到能读到数据为止。这个时候，数据被写入future，future调用done_callback，然后执行_step，然后await就有值了。在这个等待过程中，ioloop是可以干其他事的，并没有被block。

# 小结

从以上两个例子可以大概看出Python 3中native coroutine是如何并发以及如何调度的。其中，并发是依赖于ioloop，调度是依赖于send和yield。

对于future，确实和之前文章中说的那样，起到了三个作用：placeholder、call_back以及异步等待。

[learning-python]: https://www.amazon.cn/dp/1449355730/ref=sr_1_1?ie=UTF8&qid=1485009116&sr=8-1&keywords=learning+python
[PEP0492]: https://www.python.org/dev/peps/pep-0492/
[PEP0342]: https://www.python.org/dev/peps/pep-0342/
[PEP0380]: https://www.python.org/dev/peps/pep-0380/
[aiohttp-doc]: http://aiohttp.readthedocs.io/en/stable/
