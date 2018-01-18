---
layout: post
title:  "gevent基础（四）：monkey patch"
date:   2017-01-09 19:48:08 +0800
categories: python gevent
tags: python
---

gevent被称为隐式的异步，不是简单地因为它提供的接口能够像操作线程那样操作协程，而是因为使用gevent进行异步方式编程时使用的是和同步方式一样的接口，这意味着使用同步方式实现的代码可以不需要更改就可以用于异步方式。这篇文章以socket为例，来看看gevent是怎样达到这样的效果的。

# monkey patch

---

将同步方式改为异步方式的关键在于monkey patch这个模块，以socket为例，patch过后的socket就不是原来的socket了，如下所示：

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

可以看到，调用patch相关函数之后，socket模块中的一些东西已经被替换掉了。得益于Python动态语言的特点，gevent可以比较方便的替换掉Python标准库里面的一些东西。除了替换socket外，还可以替换其他东西，具体的内容可以参考monkey模块。

# gevent socket的实现

---

先来看看socket中用的最多的两个函数，recv和send：

{% highlight python %}
# gevent._socket3.py

class socket(object):

    _gevent_sock_class = _wrefsocket

    def __init__(self, family=AF_INET, type=SOCK_STREAM, proto=0, fileno=None):
        # proxy模式，得到一个_socket.socket的实例
        # 注意，_socket是CPython提供的底层module
        self._sock = self._gevent_sock_class(family, type, proto, fileno)
        ...
        _socket.socket.setblocking(self._sock, False)       # 设置成了非阻塞模式
        fileno = _socket.socket.fileno(self._sock)
        self.hub = get_hub()                                # 当前的hub
        io_class = self.hub.loop.io
        self._read_event = io_class(fileno, 1)              # 读事件
        self._write_event = io_class(fileno, 2)             # 写事件
        self.timeout = _socket.getdefaulttimeout()

    def recv(self, *args):
        while True:
            try:
                # 如果读不到东西，不会return，而是会抛出EWOULDBLOCK
                return _socket.socket.recv(self._sock, *args) 
            except error as ex:
                if ex.args[0] != EWOULDBLOCK or self.timeout == 0.0:
                    raise
            # 抛出EWOULDBLOCK后代码逻辑会执行到这里
            # 等待io，也就是挂起当前greenlet，回到loop
            self._wait(self._read_event)
            # io可读后，通过switch返回到这里，继续while

    def send(self, data, flags=0, timeout=timeout_default):
        if timeout is timeout_default:
            timeout = self.timeout
        try:
            # 如果能够发送全部或者部分数据，会return，不然抛出EWOULDBLOCK
            # 如果需要发送全部数据，使用sendall
            return _socket.socket.send(self._sock, data, flags)
        except error as ex:
            if ex.args[0] != EWOULDBLOCK or timeout == 0.0:
                raise
            # 抛出EWOULDBLOCK后等待io可写，也就是回到loop
            self._wait(self._write_event)
            # io可读后，通过switch返回到这里
            try:
                # 再尝试io写
                return _socket.socket.send(self._sock, data, flags)
            except error as ex2:
                # 如果还是EWOULDBLOCK，那就不发送了，交给上层处理
                if ex2.args[0] == EWOULDBLOCK:
                    return 0
                # 如果是奇怪的异常，抛出
                raise

{% endhighlight %}

可以看到，recv和send实现的功能和替换前socket相应接口的功能是一样的，但是，其实现方式是异步的。以recv方式为例，如果io不可读，blocking状态的socket调用recv时会被阻塞住，然后当前线程就阻塞住了，直到io可读为止。而gevent的socket调用recv时，如果io不可读，当前greenlet会被挂起，但是当前线程并未被阻塞，其他greenlet可以继续运行。最关键的是，这种不同对于接口的调用者来讲完全不感知。

除了recv和send函数外，connect、accept等函数也有类似的实现逻辑。对于接口调用者来讲，只需要按照同步的方式来编程即可，接口内部的实现可能是同步的，也可能是异步的。

# 小结

---

除了对socket的patch，gevent还提供了很多其他的patch，例如针对pipe的、针对thread的以及针对queue的等等，可以很方便地使用。

在进行monkey patch时，gevent官方推荐尽量早地patch。因为如果在import第三方module之后进行monkey patch，而第三方module在import中进行了类似local_socket=socket.socket的操作，然后用local_socket进行IO操作，那monkey patch是不会起作用的。

此外，如果第三方module没有使用Python标准库中的socket，而是用的其他实现，monkey patch也不会起作用。


[unix-network-p]: https://www.amazon.cn/dp/B01CK7JI44/ref=sr_1_4?ie=UTF8&qid=1483974145&sr=8-4&keywords=Unix+Network+Programming
[linux-p-i]: https://www.amazon.cn/dp/1593272200/ref=sr_1_1?ie=UTF8&qid=1483974499&sr=8-1&keywords=The+Linux+Programming+Interface
[seven-con]: https://www.amazon.cn/dp/1937785653/ref=sr_1_2?ie=UTF8&qid=1483974684&sr=8-2&keywords=Seven+Concurrency+Models+in+Seven+Weeks

