---
layout: post
title:  "gevent之monkey patch"
date:   2017-01-09 19:48:08 +0800
categories: python gevent
---

最近快速刷完了[UNIX Network Programming,Volume 2][unix-network-p], [The Linux Programming Interface][linux-p-i]两本书，然后继续刷 [Seven Concurrency Models in Seven Weeks][seven-con]。这本书里面首先提到了线程并发模型，加上没有提到但是也很常见的进程并发，Python的标准库对这两种已经有比较好的支持了，看源码也不会很费劲。而Functional Programming，Actor等模型Python要么是支持得不那么好，要么是没有直接的支持。但是，利用coroutine进行并发在Python社区里面是一致比较活跃的，因此打算赶紧上手。

说道coroutine，看网上提到比较多的是gevent了，此外Python3.5也基于[PEP 492][PEP492]增加了native的支持。native coroutine增加了新的关键字，并且和generator有一些历史性的纠缠，考虑到和其他版本的兼容性，还是从gevent开始吧。

gevent对[libev][libev]和[greenlet][greenlet-doc]进行了封装，得到了一个更易用的框架。虽然[官方文档][gevent-org]对gevent有了大概的介绍，但是总感觉缺那么一点意思：greenlet到底是如何切换的，又是被如何整合到异步IO里面的。中英文网页都找了一下后，发现了以下四篇博客记录了作者入门gevent的经过，对源码的分析也比较透彻，看完之后终于对gevent有了比较深的理解。

1. [python协程入门（greenlet）][greenlet-intro]，展示了greenlet最基本的使用：coroutine之间可以自己控制相互切换。而切换的关键性函数就是switch。

2. [python协程的实现（greenlet源码分析）][greenlet-source]，对greenlet的C代码进行了分析，看之前的话最好对CPython的[C extention][c-extention]已经入门。这篇主要分析了switch的实现，涉及到了汇编实现的部分。

3. [Gevent的协程实现原理][gevent-detail]，介绍了gevent如何封装greenlet，特别是一些高级接口，如join，sleep，wait等等，用起来更有线程的感觉。

4. [Gevent源码之loop的实现][gevent-loop]，介绍了gevent如何封装libev，得到自己的io loop。基本原理还是和其他io loop相同的。

除了提供高一层次的框架，gevent还提供了monkey模块，通过替换用于进行IO操作的底层对象（如socket），来让其他只能进行阻塞IO操作的第三方module兼容到gevent框架里面。本文主要来看看gevent中的socket是如何实现的，其他IO操作应该有类似实现。

{% highlight python %}
import socket
from gevent.monkey import patch_socket

print(socket.socket)
patch_socket()
print(socket.socket)

#output:
#=> <class 'socket.socket'>
#=> <class 'gevent._socket3.socket'>

{% endhighlight %}

从上面的输出结果可以看到，socket确实被替换掉了，并且根据输出，可以很方便找到源码文件的路径。下面摘录了关键性的四个函数来仔细看一下。

{% highlight python %}
# gevent._socket3.py

class socket(object):

    _gevent_sock_class = _wrefsocket

    def __init__(self, family=AF_INET, type=SOCK_STREAM, proto=0, fileno=None):
        # Take the same approach as socket2: wrap a real socket object,
        # don't subclass it. This lets code that needs the raw _sock (not tied to the hub)
        # get it. This shows up in tests like test__example_udp_server.
        self._sock = self._gevent_sock_class(family, type, proto, fileno)      # 得到一个_socket.socket的实例，proxy模式 
        self._io_refs = 0
        self._closed = False
        _socket.socket.setblocking(self._sock, False)          # 设置成了非阻塞模式
        fileno = _socket.socket.fileno(self._sock)
        self.hub = get_hub()                                   # 当前的hub
        io_class = self.hub.loop.io
        self._read_event = io_class(fileno, 1)                 # 读事件
        self._write_event = io_class(fileno, 2)                # 写事件
        self.timeout = _socket.getdefaulttimeout()

    def recv(self, *args):
        while True:
            try:
                return _socket.socket.recv(self._sock, *args) 
            except error as ex:
                if ex.args[0] != EWOULDBLOCK or self.timeout == 0.0:
                    raise
            self._wait(self._read_event)                        # 未接受完成的话挂起等待            

    def send(self, data, flags=0, timeout=timeout_default):
        if timeout is timeout_default:
            timeout = self.timeout
        try:
            return _socket.socket.send(self._sock, data, flags)
        except error as ex:
            if ex.args[0] != EWOULDBLOCK or timeout == 0.0:
                raise
            self._wait(self._write_event)   # 等待IO事件
            try:
                return _socket.socket.send(self._sock, data, flags)
            except error as ex2:
                if ex2.args[0] == EWOULDBLOCK:
                    return 0
                raise     # 操作系统已经告诉我可以send了，如果还不能send，那就抛出异常吧。

    def _wait(self, watcher, timeout_exc=timeout('timed out')):

        if watcher.callback is not None:
            raise _socketcommon.ConcurrentObjectUseError('This socket is already used by another greenlet: %r' % (watcher.callback, ))
        if self.timeout is not None:
            timeout = Timeout.start_new(self.timeout, timeout_exc, ref=False)
        else:
            timeout = None
        try:
            self.hub.wait(watcher)
        finally:
            if timeout is not None:
                timeout.cancel()

{% endhighlight %}

与Python标准库的socket实现略有不同。标准库的socket继承了_socket.socket，而gevent中的socket内置了一个_socket.socket对象，由socket提供接口供外部调用。此外，gevent将_socket.socket对象设置成了非阻塞模式。

CPython底层使用的是BSD socket interface，但是经过封装后隐藏了很多细节，特别是在non-blocking模式下。参照官方文档的描述以及标准库asyncio中的使用方式，可以判断相关接口的行为。下面就拿recv和send方法为例，看看gevent是怎样实现异步IO socket的。

对于阻塞型socket，调用recv从内核缓冲区（也就是内核为每个TCP连接维护的缓冲区）获取指定长度的内容，如果内核缓冲区中没有这么多内容，那么会阻塞当前线程，直到内核从网卡缓冲区拿到了足够的内容。对于非阻塞的socket，如果内核中没这么多数据，会抛出BlockingIOError。在gevent中，这个时候就可以调用_wait将当前coroutine挂起，将该socket的fileno放入ioloop中，等待IO事件唤醒该coroutine，最后一次性读取。

需要注意的是，recv接受args，并将args全部传入_socket.socket.recv中。第三方module使用的socket被替换成gevent的socket后，调用recv传输参数会不会产生冲突还要具体分析。

同理，对于阻塞型socket，如果内核缓冲被写满，调用send会阻塞当前线程，直到内核把数据刷到网卡的缓存后有了多余的空间。对于非阻塞socket，那就是能写多少写多少，然后立刻返回成功写入的长度。如果内核一个字节都接收不了，会抛出BlockingIOError。gevent利用_wait挂起coroutine，然后等待IO事件。

在进行monkey patch时，gevent官方推荐尽量早地patch。因为如果在import第三方module之后进行monkey patch，而第三方module在import中进行了类似local_socket=socket.socket的操作，然后用local_socket进行IO操作，那monkey patch是不会起作用的。此外，如果第三方module没有使用Python标准库中的socket，而是用的其他实现，monkey patch也不会起作用。


[greenlet-intro]: http://blog.csdn.net/fjslovejhl/article/details/38821673
[greenlet-source]: http://blog.csdn.net/fjslovejhl/article/details/38824963
[gevent-detail]: http://blog.csdn.net/fjslovejhl/article/details/39007831
[gevent-loop]: http://blog.csdn.net/fjslovejhl/article/details/39153861
[unix-network-p]: https://www.amazon.cn/dp/B01CK7JI44/ref=sr_1_4?ie=UTF8&qid=1483974145&sr=8-4&keywords=Unix+Network+Programming
[linux-p-i]: https://www.amazon.cn/dp/1593272200/ref=sr_1_1?ie=UTF8&qid=1483974499&sr=8-1&keywords=The+Linux+Programming+Interface
[seven-con]: https://www.amazon.cn/dp/1937785653/ref=sr_1_2?ie=UTF8&qid=1483974684&sr=8-2&keywords=Seven+Concurrency+Models+in+Seven+Weeks
[PEP492]:https://www.python.org/dev/peps/pep-0492/
[c-extention]:https://docs.python.org/3/extending/index.html
[libev]:http://pod.tst.eu/http://cvs.schmorp.de/libev/ev.pod
[greenlet-doc]:https://greenlet.readthedocs.io/en/latest/
[gevent-org]:http://www.gevent.org/
