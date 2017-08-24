---
layout: post
title:  "博客摘要"
date:   2017-08-24 21:00:00 +0800
categories: python
tags: other
---

# Asynchronous Python and Databases

---

[http://techspot.zzzeek.org/2015/02/15/asynchronous-python-and-databases/](http://techspot.zzzeek.org/2015/02/15/asynchronous-python-and-databases/)：SQLAlchemy作者对于Python异步IO（主要是asyncio）用于数据库查询（CURD）的看法。

异步IO适用于需要并发地保持**大量**且**比较慢**的TCP链接、只需要相对偶尔地发送／接受数据的应用场景，例如HTTP server（可能有比较慢的客户端）、聊天程序、消息队列。在这样的场景下，让线程去等待IO不是那么有效率。Javascript所在的前端领域用的主要也是异步IO，也是类似的原因。

node.js在后端也广泛使用异步IO，这主要得益于extremely performant JIT-enabled engine，以及并发的连接数不是很多（通常在10～50）。

对于让SQLAlchemy支持asyncio的呼吁，作者持比较开放的态度，表示乐意增加这样的支持，但是作者比较怀疑人们是否真的希望使用这种异步IO的模式。在HTTP server、聊天程序中，异步IO确实是更好的选择，但是在操作数据库方面（CRUD），特别是操作本地数据库（网络耗时低），情况不一定是人们想象的那样。

1. 对于C和Java实现的应用而言，等待IO的时间会比执行代码的时间多很多，但是Python非常慢，所以Python执行代码的时间并没有想象中的那么短（在CRUD的场景下）。比较C实现的DBAPI（MySQLdb）和Python实现的DBAPI（pymysql），测试INSERT和SELECT总体的执行时间，在有网络通信的情况下，Python版本的耗时比C版本的多35%，在使用本地数据库的情况下，Python版本耗时是C版本的9倍。再考虑Python版本的IO时间和CPU时间，IO占1/3，CPU占2/3，本地通信和网路通信均是如此。这是在仅仅测试DBAPI的情况下，实际的应用程序当中还有很多其他的逻辑，CPU用时占比只会增不会减。

2. asyncio用到了yield from + StopIteration来获得携程的返回值，耗时是普通return的6倍。也就是说，asyncio是比较低效率的，深度使用会降低响应速度。使用psycopg2、aiopg、psycogreen来对比thread、gevent、asyncio在执行SELECT的总体速度，thread比gevent略快（22k r/sec vs 18k r/sec for Python 2.7.8），asyncio最慢（8k r/sec for Python 3.4.1）。

总体而言，异步IO和线程模型各有优缺点，并不是某种一定好于另外一种。但是对于数据库的CRUD操作，asyncio可能反而会降低性能，这种情况线程池模型更适合。

