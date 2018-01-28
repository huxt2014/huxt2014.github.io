---
layout: post
title:  "MySQL/Mariadb：internal lock浅析"
date:   2017-01-18 19:48:08 +0800
categories: mysql
tags: web
---

Mariadb内部会有多个线程执行任务来满足高并发的需求，internal lock用于协调这些线程的工作，保证数据的正确性。之所以称为internal，是因为对这些锁的控制完全由Mariadb掌控，不依赖其他程序。这篇文章主要记录InnoDB中internal lock的相关用法。

# consistent read和repeatable read

---

internal lock的存在主要是为了保证数据库transaction之间的隔离性，因此在讨论internal lock之前先来看看InnoDB默认配置下采用的隔离方案：consistent read和repeatable read。

repeatable read指的是SQL规范中的隔离的等级，其要求当前transaction仅能够读到其他transaction已经commit的内容，并且只要读了一次，后边再读的内容和第一次读到的一样（其他隔离等级参考SQL规范）。consistent read指的是InnoDB中实现select的一种方式，也就是使用snapshot来表示select的查询结果。

具体来讲，在默认配置下使用select进行查询时，InnoDB内部使用snapshot来保存这个结果，且这个snapshot在当前transaction中第一次使用select时生成。不管其他transaction进行了什么操作，当前transaction的后续select得到的结果都是和这个snapshot一致的。

这种模式的特点是在进行select操作的时候不需要使用lock（也就是不会被block），并且保持了一定的隔离性。但是，有两种特殊的情况是需要特别注意的:

1. 当前transaction进行了 a）update操作 b）select操作。如果另外一个transaction在a）和b）之间对数据做了一些更改，那么这些更改在当前transaction的b）中是可见的。这种情况下，当前transaction在b）中查询到的状态是一个数据库中没有存在过的状态。

2. 当前transaction进行了 a）select操作 b）update操作。如果另外一个transaction在a）和b）之间insert了一些记录，那么当前transaction在b）的操作能够影响到这些新增记录，并且这些新增记录在当前transaction中可能变为可见的。也就是说，在当前transaction中consistent read对DML是失效的。


# internal lock基本介绍

---

为了达到更高级别的隔离，InnoDB提供了row-level lock。相对于单纯锁掉整张表，这种更细粒度的锁可以提供更高的并发性。与之对应的代价是，如果使用不当更易造成死锁。在InnoDB中，与row-level lock一同起作用的，还有一种table-level lock：intention lock。虽然也被称为锁，但intention lock并不是真正的锁，它仅仅起到一个标示作用。

对于row-level lock，官方文档是这样解释的：InnoDB提供了两种row-level lock，shared(S) lock 和 exclusive(X) lock。对于一条记录，当一个transaction获得了它的S lock，另外一个transaction仅能获得它的S lock；如果一个transaction获得了它的X lock，另外一个transaction既不能获得X lock，也不能获得S lock。

对于intention lock，也有两种类型：intention shared(IS) lock 和intention exclusive(IX) lock。对于intention lock的使用规则，官方文档是这样解释的：当一个transaction需要获得某个表中的S lock时，必须先获得这张表的IS lock 或者 IX lock；当一个transaction需要获得某个表中的X lock时，必须先获得这张表的IX lock。

实际上，IS lock 和IX lock 并没有锁表：当一个transaction已经获得了一张表的IX lock或IS lock时，另外一个transaction可以获得这张表的IX lock或者IS lock中的任意一个。Intention lock只起到一个标示作用：某个transaction在某张表中已经获得／将要获得某些行对应类型的row-level lock。此外，当transaction获得了intention lock时，该transaction访问表中的记录时，会加上与intention lock同类型的row-level lock。

S lock 和X lock之间的互斥性是显而易见的，intention lock之间以及intention lock和row-level之间没有互斥性。对于互斥性，官方文档里面那张奇怪的表格貌似没表述清楚，具体解释可以[参照这个][description1]。


# 获取lock

---

intention lock相对比较简单：select ... lock in share mode会获得IS lock，select ... for update 会获得IX lock，update、insert 和 delete都会获得IX lock。这里主要来看一下row-level lock。

row-level lock作用在index上，可以细分为三种：

1. record lock，锁住一条记录及其对应的index。

2. gap lock，锁住index上的一个gap。所谓的gap，就是index中值与值之间的空隙。举例而言，在一个整数类型的列上建了索应，则(1,20)可以是一个gap。如果这个gap被加上了锁，那么(1,20)之间无法插入新的记录。除了两个值之间，类似于(负无穷,1)和(20,正无穷)也算是一个gap。

3. next-key lock，相当于一个record lock加上一个gap lock，其中gap lock加在对应record之前的一个gap上。例如，一个index上有10、13、20这么几个值，next-key lock可以是：(负无穷,10]，(10,13]，(13, 20]，(20,正无穷)。

与intention lock类似，在默认模式下，select ... lock in share mode会获得S lock，select ... for update以及update、delete、insert会获得X lock。更细节的规则以及例子如下所示：

{% highlight text %}
1. 较早版本中internal lock的实现使用到了Pthreads mutex，现在使用的是gcc的atomic
   memory access机制。
2. 在使用范围查找或者在非unique index类型的索引上查找的时候，扫描过的范围都会被加上锁。如果
   查询过程扫描了整张表，那么整张表都会被锁住；如果只扫描了部分表，那么一部分会被锁住。
3. 如果扫描发生在非clustered index上，那么对应的clustered index上也会被加上锁。
4. 如果在unique index类型的索引上使用=进行查询，并且能够准确命中（没有进行范围查找）一条记
   录，那么这条记录上会被加上record lock。其他情况下通常使用的是next-key lock，并且会在最
   后一条记录之后加上gap lock。
5. 通常情况下，insert只会对新增的记录添加record lock。如果另外一个transaction已经新增了
   一条记录且加上了record lock，当前transaction希望在同一位置插入记录时，会引起
   duplicate-key error，此时当前的transaction转而获得该记录的S lock。等另外一个
   transaction释放record lock后，当前transaction会尝试获得该记录的X lock。这种分步完成
   的动作可能造成死锁。
6. 如果使用insert ... on duplicate key update，在触发duplicate-key error时会尝试获
   得X next-key lock，而不是获得S lock。
7. 如果rollback了整个transaction，那么所有的锁都会被释放。如果因为某些错误导致单条查询失
   败，因为这条查询而添加的锁可能不会被全部释放。
8. 举例：如果在index(id)上有值10、 20，那么select ... where id > 15 for update 会
   锁住(10, 20]、(20,正无穷)。select ... where id < 15 for update 会锁住
   (负无穷,10]、(10,20)。
9. 举例：如果在index上有值10、 20，那么select ... where id in (10, 13, 20) for
   update 会锁住10以及(10,20]，id in (15, 20)会锁住(10,20]，id in (13, 15)会锁住
   (10,20)。

{% endhighlight %}

# 避免死锁

---

细粒度的锁可以提高并发性，但是使用不当也更可能造成死锁。如上文所说，在InnoDB中哪怕插入一条记录都有可能造成死锁（因为插入操作不是“atomic”）。虽然InnoDB有自动检测死锁的机制，能够自动打断死锁将transaction rollback，但是频繁地发生这样的情况也会造成性能下降，所以正确做法还是要尽量避免死锁的发生。以下列出一些减少死锁发生频率、降低死锁影响的小技巧：

{% highlight text %}
1. 能不用锁就不用，默认情况下有consist read。
2. 在transaction一开始就应该获取所有的锁，尽量避免分步进行，例如先获得一些记录的锁再获得另外
   一些记录的锁，或者先获得S lock然后再获得X lock。
3. 如果无法一次性获得所有锁，则按全局一致的顺序依次获得所有锁。
4. 让transaction尽可能地短，仅早commit。
5. 写应用层逻辑时让transaction可以随时rollback然后重来。
6. 妥善建立索引，使得查询时扫描更少的数据量（也就是锁住更少的记录）。
7. show engine innodb status 命令可以查看transaction中锁的情况，设置
   innodb_status_output_locks = 1时可以看到lock的详细情况。Mariadb在10.0.14版本才加入
   了innodb_status_output_locks配置项。
8. 必要情况下可以开启innodb_print_all_deadlocks，所有的死锁都会记录在error log中。
9. 某些情况下可以降低隔离等级，例如降为read committed。
10. 某些情况下可以考虑把整个表都锁掉。

{% endhighlight %}


# 应用举例

---

1. 对于两张表parent和child，如果我们select出一条parent判断其是存在的，然后若需要对其插入一条child记录，那么select需要获得S lock，如果插入child的同时需要修改parent，那么select需要获得X lock。



[description1]: http://bugs.mysql.com/bug.php?id=63665
