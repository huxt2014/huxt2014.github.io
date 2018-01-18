---
layout: post
title:  "SQLAlchemy：让mysqlclient支持gevent"
date:   2017-01-10 19:48:08 +0800
categories: python gevent
tags: web
---

SQLAlchemy支持的[MySQL driver][driver-list]有很多，其中 mysql-python是默认选择。而mysql-python分为了两个module，[mysqlclient][mysqlclient]支持Python 3，支持Python 2的[MySQL-python][MySQL-python]已经不提供功能方面的更新了。这篇文章主要看看如何让mysqlclient支持gevent进行异步io。

# 同步io的效果

---

首先，构造一个测试用的数据库，往里面写足够量的数据。下面的查询语句会对全表进行扫描，查询一次用时1.5s左右。

{% highlight python %}

def do_query(con):
    con.execute('select count(*) from test where flow > 5000')

#=> 0:00:01.541861
{% endhighlight %}

接下来，打上monkey patch，创建四个连接，用四个coroutine同时执行查型语句，执行结果如下，总用时在6s左右。从结果可以推断，即使调用了monkey_patch，实际上并没有达到异步IO的效果。

{% highlight python %}
from datetime import datetime

import gevent
from gevent.monkey import patch_all
patch_all()
from sqlalchemy import create_engine

engine = create_engine(get_database_url(), encoding='utf-8')
gs = []

for i in range(4):
    gs.append(gevent.spawn(do_query, engine.connect()))

begin_time = datetime.now()
gevent.joinall(gs)
end_time = datetime.now()

print(end_time-begin_time)

# output:
#=> 0:00:05.913082
{% endhighlight %}

# 转化为异步io

---

mysqlclient遵循[PEP 249][pep249]规范，稍微搜索一下源码后可以找到负责与数据库进行交互的关键部分，如下所示：
{% highlight python %}
# MySQLdb/connections.py

class Connection(_mysql.connection):
    """MySQL Database Connection Object"""

    default_cursor = cursors.Cursor
    waiter = None
    def __init__(self, *args, **kwargs):
        #省略 ... 
        self.waiter = kwargs2.pop('waiter', None)
        #省略...
    
    def query(self, query):
        # Since _mysql releases GIL while querying, we need immutable buffer.
        if isinstance(query, bytearray):
            query = bytes(query)

        # 这个waiter怎么感觉和gevent的waiter长得很像
        if self.waiter is not None:
            self.send_query(query)
            self.waiter(self.fileno())
            self.read_query_result()
        else:
            _mysql.connection.query(self, query)

{% endhighlight %}

初步推断waiter部分与gevent有点关联，再搜索一下发现mysqlclient官方github上使用gevent的[代码示例][waiter-sample]，那么可以确认这就是支持gevent的关键了。SQLAlchemy不推荐应用直接操作connection实例，而是维护了自己的engine和connection pool，waiter可以通过相关的接口传递到connection中，如下所示：

{% highlight python %}

def waiter(fd):
    hub = gevent.hub.get_hub()
    hub.wait(hub.loop.io(fd, 1))

engine = create_engine(get_database_url(), encoding='utf-8',
                       connect_args={'waiter': waiter})
gs = []

for i in range(4):
    gs.append(gevent.spawn(do_query, engine.connect()))

#=> 0:00:01.420103

{% endhighlight %}

修改代码后，重新执行程序，四次查询的时间在1.5s左右。从查询耗时上可以看出，确实是进行了异步IO。

扫了一边mysqlclient的源码，基本可以确认支持异步化的操作就这一处。至于send_query与read_query_result，其底层用的是mysql的C API，即使API内部能够让当前线程释放CPU，但大概率是阻塞当前线程中的其他协程的。换句话说，只有send_query与read_query_result之间能够进行异步等待，如果查询语句／查询结果过长，发送语句／读取结果的过程也是同步的。

# 小结

---

[这个回答][orm-syncio]里面解释了为什么在需要显式调用callback的异步框架里面使用 ORM会力不从心，主要原因是在复杂系统中，开发者很难精确地知道每个IO操作所在的位置并利用callback使其异步执行。一旦漏掉了某个IO操作，那么这个IO阻塞住的话，会阻塞当前线程正在处理的所有请求。

即使知道了系统中所有IO的位置，很多时候也无法改变（没有修改API底层实现的权限）或者很难去改变（顺序执行的代码和callback风格的代码无法轻易兼容）。在操作数据库的时候，虽然可以从DPAPI层面将IO异步化来改善这种情况，但是也不能根本性地解决问题（例如Lock在同步和异步中使用的是不一样的实现）。

gevent的优势在于，monkey_patch几乎可以将所有类型的同步IO操作都替换成异步IO，并且保持接口不变。通过这种方式，上层应用依旧调用的是同步的IO接口，但是接口底层已经被异步化了，这就是implicit asynchronous。



[driver-list]: http://docs.sqlalchemy.org/en/latest/dialects/mysql.html
[mysqlclient]: https://mysqlclient.readthedocs.io/en/latest/
[MySQL-python]:https://pypi.python.org/pypi/MySQL-python/1.2.5
[pep249]:https://www.python.org/dev/peps/pep-0249/
[waiter-sample]: https://github.com/PyMySQL/mysqlclient-python/blob/master/samples/waiter_gevent.py
[orm-syncio]: http://stackoverflow.com/questions/16491564/how-to-make-sqlalchemy-in-tornado-to-be-async
