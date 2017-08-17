---
layout: post
title:  "Gunicorn的特性与实现"
date:   2017-08-08 21:00:00 +0800
categories: web
tags: web
---

Gunicorn(Green Unicorn)是一个Python实现的WSGI HTTP Server，它实现了pre-fork worker model，并且支持在不停止服务的情况下更新配置和升级。这篇文章通过对Gunicorn的学习，来看看这些特性可以如何实现。

# pre-fork worker model

---

所谓的pre-fork worker model通常是指在基于master-worker的工作模式下，master进程在客户端请求到来之前已经fork出了若干个worker进程，客户端请求到达时由worker进程进行处理，master进程主要负责worker进程的管理。在Gunicorn中，这个模型的实现主要由三个类来支持：Application、Arbiter、Worker。Application主要负责配置的加载和存储，Arbiter实现了管理worker的主要逻辑，而Worker负责接收socket请求并且调用wsgi_app（Django、Flask等）进行处理。

在启动过程中，master进程首先会加载完成各类配置信息，包括wsgi_app的路径，基本上不存在某些配置信息是动态（fork之后）加载的。socket bind的动作也在fork之前完成，这样fork之后worker进程就可以使用这个socket调用accept了。此外，创建Worker实例的过程也是在fork之前完成，因为Worker实例中存在一些master进程与worker进程进行通信所需要的数据，需要留存在master进程中。在这之后就可以fork子进程了。

fork之后，worker进程才会依照路径加载wsgi_app，然后调用accept开始处理请求。可以说，master进程中不带有wsgi_app和客户端请求的任何信息。

# 信号队列

---

master进程的管理动作主要是通过signal来驱动的，例如worker意外终止后会发送SIGCHLD，重新加载配置可以发送HUP等。Gunicorn只有为SIGCHLD信号直接设置了handler，其余的信号都是通过信号队列来完成。

{% highlight python %}
class Arbiter(object):

    def start(self):
        ...
        self.init_signals()
        ...

    def init_signals(self):
        """\
        Initialize master signal handling. Most of the signals
        are queued. Child signals only wake up the master.
        """

        ...
        self.PIPE = pair = os.pipe()
        ....
        # initialize all signals
        [signal.signal(s, self.signal) for s in self.SIGNALS]
        signal.signal(signal.SIGCHLD, self.handle_chld)

    def signal(self, sig, frame):
        if len(self.SIG_QUEUE) < 5:
            self.SIG_QUEUE.append(sig)
            self.wakeup()

    def run(self):
        "Main master loop."
        ...
        try:
            ...
            while True:
                sig = self.SIG_QUEUE.pop(0) if len(self.SIG_QUEUE) else None
                ...
                signame = self.SIG_NAMES.get(sig)
                handler = getattr(self, "handle_%s" % signame, None)
                ...
                handler()
                ...
        except Exception:
            ...

{% endhighlight %}

从代码中可以看到，收到信号时，信号会被放到pipe中，在进程的主循环中会从pipe中取信号，并找到对应的handler。如果直接对signal设置handler，那么这种handler需要reentrant，通过队列的方式，就可以避免这个限制了。

# 请求分流

---

Master进程在fork之前初始化socket并调用bind和listen，之后基本上就不用管了。fork之后子进程会自动得到父进程的file descriptor，所以worker进程可以直接使用这些socket调用accept，操作系统保证了多个进程可以同时调用accept。所以，master进程对客户端的请求一无所知，全部由worker进程进行处理。

worker进程调用accept得到客户端请求之后，处理逻辑根据不同类型的worker而不同。sync类型的worker使用的是单线程模型，调用accept和处理客户端请求的操作在同一个线程中执行。gthread类型的worker使用的是线程池模型，主线程poll之后调用accept，用concurrent.futures的线程池处理客户端请求。

除了上述两种worker，Gunicorn还支持其他并发模型的worker，包括gevent、Python 3 AsyncIO、Tornado
等等。

# 保活

---

worker进程由于异常原因可能会卡死（例如内存耗尽后Python继续申请内存时），如果不kill掉然后重启一个就会造成worker进程依旧在正常工作的假象。一个常用的解决方案是心跳机制，即worker隔一段时间通知master自己还活着，如果master间隔太久没收到worker的心跳，那么就可以kill掉然后重启一个worker了。至于具体用什么方式进行通信，那就可以视情况而定了。

如果worker和master处于不同的机器上，那么可以用socket来实现这种通信。但是Gunicorn中master和worker处于同一台机器中，所以使用了一个相对而言更简单的方案：写文件。具体来说，每个worker进程都关联到一个文件，worker每隔一段时间会写一次文件，master会隔一段时间去检查文件的修改时间，通过这种方式也可以达到通信效果。

# 配置更新

---

Gunicorn可以在不停止服务的情况下更新配置。所谓的不停止服务，具体来讲就是处于listen状态的socket能够一直正常工作（能够调用accept），不会因为更新配置而断开，并且worker进程中的客户端请求不会因为更新配置而异常终止。Gunicorn能够比较轻松地做到这一点，得益于良好的配置管理以及master和worker的低耦合。

Gunicorn所有的配置管理都位于master进程中，不论是load还是reload，都是由master进程在执行fork前完成并且存储在master进程的内存当中。当执行fork时，这些配置信息自然就带到了worker进程中了。所以，在更新配置的时候，只要在master中执行reload，然后fork出新的worker进程，再停掉老的worker进程就可以了。在这个过程中，并不需要对socket做额外的操作。

如果wsgi_app做了更改需要升级，也可以通过这个过程来达到不停止服务进行升级的目的，因为wsgi_app完全在worker进程中完成加载。

# 热重启／升级

---

除了更改配置，Gunicorn也可以在不停止服务的情况下对自己进行升级，也就是重启master进程。这里面的关键点在于让两个master进程（一新一旧）同时工作，然后kill其中一个，并且这个过程不会造成socket的中断。

为了让socket不中断，就需要让两个master进程同时使用一个listen状态的socket。如果只是简单启动第二个master进程，实现起来不是很方便。因为当一个socket与IP:PORT绑定后，默认情况下是不允许另外一个进程将另外一个socket对同一个IP:PORT进行绑定的。gunicorn的解决方案是通过发送信号，通知master fork出第二个master，然后再通过exec执行新的代码。

新的master进程exec后依旧会得到父进程的fd，所以可以正常使用处于listen状态的socket。但是由于清掉了原来的内存，新的master进程并不知道到底使用哪一个fd。Gunicorn通过环境变量来传递socket fd的信息，那么新的master进程就可以找到socket对应的fd了。通过这样的方式，两个master进程及其各自对应的worker进程都可以同时使用一个listen状态的socket并且正常工作了，之后可以kill掉两个master进程中的任意一个。

