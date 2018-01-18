---
layout: post
title:  "SQLAlchemy：QueuePool浅析"
date:   2017-01-16 19:48:08 +0800
categories: python sqlalchemy
tags: web
---


这篇文章主要记录一下SQLAlchemy中连接池的一些特性是如何实现的。

# lazy initialization 和 overflow

---

对于一个最简单的pool，例如python3标准库queue.Queue，一般在初始化的时候会顺带初始化里面的对象，这种初始化可以称之为eager initialization。此外，pool的大小一般是确定的。在SQLAlchemy的QueuePool中，针对以上两个行为（初始化和确定大小）提供了不同的方式：lazy initialization和 overflow。

所谓的lazy initialization，就是在初始化pool的时候，不初始化里面的对象，在有需求的时候才初始化。并且，即使将pool的size设置为10，如果实际中最大使用量为5，那么poll里面的对象也只有5个，而不是10个。

所谓的overflow，是指当从pool中取出的对象数量已经达到pool size了，pool依旧可以继续提供一定数量的对象，当多余的对象尝试放回到pool的时候会被销毁。

{% highlight python %}
# sqlalchemy/pool.py

class QueuePool(Pool):

    def __init__(self, creator, pool_size=5, max_overflow=10, timeout=30,
                 **kw):
        Pool.__init__(self, creator, **kw)
        self._pool = sqla_queue.Queue(pool_size)
        self._overflow = 0 - pool_size
        self._max_overflow = max_overflow
        self._timeout = timeout
        self._overflow_lock = threading.Lock()
        
    def _do_get(self):
        use_overflow = self._max_overflow > -1

        try:
            wait = use_overflow and self._overflow >= self._max_overflow
            return self._pool.get(wait, self._timeout)
        except sqla_queue.Empty:
            if use_overflow and self._overflow >= self._max_overflow:
                if not wait:
                    return self._do_get()
                else:
                    raise exc.TimeoutError(
                        "QueuePool limit of size %d overflow %d reached, "
                        "connection timed out, timeout %d" %
                        (self.size(), self.overflow(), self._timeout))

            if self._inc_overflow():
                try:
                    return self._create_connection()
                except:
                    with util.safe_reraise():
                        self._dec_overflow()
            else:
                return self._do_get()

    def _do_return_conn(self, conn):
        try:
            self._pool.put(conn, False)
        except sqla_queue.Full:
            try:
                conn.close()
            finally:
                self._dec_overflow()

{% endhighlight %}

以上列出了关键性的源码，其中底层存储对象的容器是collections.deque。可以看到QueuePool在初始化的时候并没有初始化里面的对象，而是在_do_get方法中完成。此外，_do_get和_do_return_conn提供了overflow的支持。

# facade for DBAPI connection

---

QueuePool里面存储的对象是数据库连接，而数据库连接中常见的异常就是连接丢失/连接失效。对于这种异常，通常做法是重新连接，但是“重新连接”这一逻辑放在哪里呢？OOP中有Single responsibility principle，在“重新连接”这个逻辑中，SQLAlchemy中实践这一原则的方式是为DBAPI connection提供一个facade：_ConnectionRecord。

在QueuePool中实际存的是_ConnectionRecord对象，一个_ConnectionRecord对象内置了一个DBAPI connection。如果DBAPI connection失效了，_ConnectionRecord负责重新连接。所以在一个_ConnectionRecord对象的生命周期当中，可能会使用到多个DBAPI connection。

# return on dereference

---

当connection从pool取出后，应该要保证使用完之后放回pool，这一逻辑可以在业务代码中显式地保证，但是SQLAlchemy还提供了一个隐式的保证。

{% highlight python %}

In [9]: engine.pool._pool.qsize()
Out[9]: 1

In [10]: conn = engine.connect()

In [11]: engine.pool._pool.qsize()
Out[11]: 0

In [12]: del conn

In [13]: engine.pool._pool.qsize()
Out[13]: 1

{% endhighlight %}

在IPython中作了如上操作，可以看到conn并没有调用close方法，但是conn依旧回到了pool中，这是因为SQLAlchemy使用了[weekref][weekref]中的callback。根据CPython的gc机制，当weakref引用的对象即将被回收时会调用callback，callback将连接放入到pool中。

weakref引用的对象并不是_ConnectionRecord，而是_ConnectionFairy。_ConnectionFairy作为facade，实现了return on dereference这一逻辑。

# high level proxy

---

在业务代码中，需要调用engine.connect()得到数据库连接，利用数据库连接对数据库进行操作，而这个连接是sqlalchemy.engine.base.Connection对象。这个对象提供了高一层次访问数据库的接口，相当于又加了一个facade。

所以，对于业务代码中使用的连接，有这样的facade逻辑：sqlalchemy.engine.base.Connection -> sqlalchemy.pool._ConnectionFairy -> sqlalchemy.pool._ConnectionRecord -> DBAPI connection。

另外一个值得一提的是，engine.connect()中所调用的大多数函数都reentrant，其中的例外只有对QueuePool的操错和对一个全局set对象的操作。对于QueuePool的使用已经利用锁进行线程同步了，而set默认是线程安全的。

# 小结

---

SQLAlchemy的QueuePool在初始化的时候不需要初始化里面的数据库连接，且在某一时刻能够checkout的连接数可以超过Pool的size；对于数据库连接失效的处理可以添加在Pool的checkin和checkout逻辑中，但是SQLAlchemy将这一操作单独放在_ConnectionRecord类中；数据库连接使用完成后需要放回到Pool中，SQLAlchemy使用了weakref来保证这一过程。

[weekref]: https://docs.python.org/3.6/library/weakref.html
