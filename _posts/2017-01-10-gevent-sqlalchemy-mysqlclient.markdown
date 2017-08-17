---
layout: post
title:  "让SQLAlchemy支持gevent：mysqlclient异步IO"
date:   2017-01-10 19:48:08 +0800
categories: python gevent
tags: web
---

前一篇文章提到，monkey patch不能将任意的同步IO替换为异步IO，那问题来了，monkey patch能替换数据库IO么。这篇文章主要来看一下如何将SQLAlchemy的数据库IO替换成gevent异步IO。

SQLAlchemy支持的[MySQL driver][driver-list]有很多，其中 mysql-python是default DBAPI。而mysql-python分为了两个module，[mysqlclient][mysqlclient]支持Python 3，支持Python 2的[MySQL-python][MySQL-python]已经不提供功能方面的更新了。所以，这里主要考虑mysqlclient。

首先，先构造一个测试用的数据库，往里面写足够量的数据。下面的查询语句会对全表进行扫描，查询一次用时1.5s左右。

{% highlight python %}
from datetime import datetime

import gevent
from sqlalchemy import create_engine

def do_query(con):
    con.execute('select count(*) from test where flow > 5000')

engine = create_engine(get_database_url(), encoding='utf-8')    
g = gevent.spawn(do_query, engine.connect())

begin_time = datetime.now()
g.join()
end_time = datetime.now()

print(end_time-begin_time)

# output:
#=> 0:00:01.541861
{% endhighlight %}

接下来，打上monkey patch，创建四个连接，用四个coroutine同时执行查型语句，执行结果如下，总用时在6s左右。从结果可以推断，进行的依旧是同步IO，并没有达到异步IO的效果。网上搜了一下如何让SQLAlchemy支持gevent，貌似没有现成的解决方案，略感忧伤，看来又只能从源码着手了。

{% highlight python %}
from datetime import datetime

import gevent
from gevent.monkey import patch_all
patch_all()
from sqlalchemy import create_engine

def do_query(con):
    con.execute('select count(*) from test where flow > 5000')

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

SQLAlchemy用的是DBAPI访问数据库，而DBAPI是遵循[PEP 249][pep249]规范的。此外，访问数据库的逻辑应该是send query，然后等待结果。结合这两点，可以很容易定位到关键源码。源码列举如下，省略了非关键的部分。

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
        if self.waiter is not None:       # 这个waiter怎么感觉和gevent的waiter长得很像
            self.send_query(query)
            self.waiter(self.fileno())
            self.read_query_result()
        else:
            _mysql.connection.query(self, query)

{% endhighlight %}

源码里面的waiter怎么看怎么感觉和gevent会有点关系啊，不然再上网搜搜看？果然，用mysqlclient+waiter关键词搜到了想要的东西：mysqlclient官方github上使用gevent的[代码示例][waiter-sample]。看来之前搜索的姿势不对么。

从代码示例上可以看到，维护mysqlclient的社区已经考虑到了兼容gevent，那接下来就是看怎样将waiter通过SQLAlchemy传递到connection实例了。

SQLAlchemy不推荐应用直接操作connection实例，而是维护了自己的engine和connection pool，根据使用经验，SQLAlchemy应该会提供接口，允许应用在初始化connection实例的时候提供额外的参数。没错，稍微找了一下后，就是在create_engine函数中：`connect_args¶ – a dictionary of options which will be passed directly to the DBAPI’s connect() method as additional keyword arguments. `

{% highlight python %}
from datetime import datetime

import gevent
from sqlalchemy import create_engine

def waiter(fd):
    hub = gevent.hub.get_hub()
    hub.wait(hub.loop.io(fd, 1))

def do_query(con):
    con.execute('select count(*) from test where flow > 5000')

engine = create_engine(get_database_url(), encoding='utf-8',
                       connect_args={'waiter': waiter})
gs = []

for i in range(4):
    gs.append(gevent.spawn(do_query, engine.connect()))

begin_time = datetime.now()
gevent.joinall(gs)
end_time = datetime.now()

print(end_time-begin_time)

# output:
#=> 0:00:01.420103

{% endhighlight %}

修改代码后，重新执行程序，四次查询的时间在1.5s左右。从查询耗时上可以看出，确实是进行了异步IO。虽然确认了send query完成后可以将当前coroutine挂起，等待查询结果，但是还是有必要确认一下其他的操作是不是可以进行异步操作。

扫了一遍源码，发现waiter只出现在上述一个函数中。可以推断，对于mysqlclient而言，确实只有上述的这一步骤可以进行异步处理，其他操作均是同步。例如，即使query比较长，发送的时候也是同步发送的，即使查询结果数据量很大，读取也是同步的。只有在send query完成，等待数据返回时才能够异步处理。对于其他driver，可能会有不同的实现方式，需要具体分析。

以上只涉及最基础的异步查询，在使用SQLAlchemy开发应用时，一般都会使用到connection pool，SQLAlchemy已有的pool是否可以直接支持coroutine的并发，是下一步需要确认的。


补充：

[这个回答][orm-syncio]里面解释了为什么在需要显式调用callback的异步框架里面使用 ORM会力不从心，主要原因是在复杂系统中，开发者很难精确地知道每个IO操作所在的位置并利用callback使其异步执行。一旦漏掉了某个IO操作，那么这个IO阻塞住的话，会阻塞当前线程正在处理的所有请求。

即使知道了系统中所有IO的位置，很多时候也无法改变（没有修改API底层实现的权限）或者很难去改变（顺序执行的代码和callback风格的代码无法轻易兼容）。在操作数据库的时候，虽然可以从DPAPI层面将IO异步化来改善这种情况，但是也不能根本性地解决问题（例如Lock在同步和异步中使用的是不一样的实现）。

gevent的优势在于，monkey_patch几乎可以将所有类型的同步IO操作都替换成异步IO，并且保持接口不变。通过这种方式，上层应用依旧调用的是同步的IO接口，但是接口底层已经被异步化了，这就是implicit asynchronous。



[driver-list]: http://docs.sqlalchemy.org/en/latest/dialects/mysql.html
[mysqlclient]: https://mysqlclient.readthedocs.io/en/latest/
[MySQL-python]:https://pypi.python.org/pypi/MySQL-python/1.2.5
[pep249]:https://www.python.org/dev/peps/pep-0249/
[waiter-sample]: https://github.com/PyMySQL/mysqlclient-python/blob/master/samples/waiter_gevent.py
[orm-syncio]: http://stackoverflow.com/questions/16491564/how-to-make-sqlalchemy-in-tornado-to-be-async
