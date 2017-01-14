---
layout: post
title:  "MySQL InnoDB内部锁(internal lock)浅析"
date:   2017-01-14 19:48:08 +0800
categories: mysql
---

# 基本介绍
在应用开发范围内讨论并发时，人们主要讨论的是各类并发模型以及每种并发模型适用的通信和同步方式。当应用需要数据库支持时，数据库的并发访问就是一个不可回避的问题。目前工作中主要用的是MySQL，最近也是打算把并发的坑慢慢填起来，所以在学习并发模型的同时，也一并把MySQL的并发了解一下。InnoDB是 MySQL的默认引擎，也是官方推荐的通用型引擎，所以从这个开始吧。

InnoDB提供了row-level lock，相对于单纯锁掉整张表，更细粒度的锁可以提供更高的并发性。与之对应的代价是，如果使用不当更易造成死锁。在InnoDB中，与row-level lock一同起作用的，还有一种table-level lock：intention lock。虽然也被称为锁，但intention lock并不是真正的锁，它仅仅起到一个标示作用。

#### row-level lock

对于row-level lock，官方文档是这样解释的：InnoDB提供了两种row-level lock，shared(S) lock 和 exclusive(X) lock。对于一条记录，当一个transaction获得了它的S lock，另外一个transaction仅能获得它的S lock；如果一个transaction获得了它的X lock，另外一个transaction既不能获得X lock，也不能获得S lock。

从规则上看，这和多线程中read-write lock的规则类似，所以可以这样理解：对于transaction的并发，InnoDB内部为每条记录维护了一个read-write lock。

#### intention lock

对于intention lock，也有两种类型：intention shared(IS) lock 和intention exclusive(IX) lock。对于intention lock的使用规则，官方文档是这样解释的：当一个transaction需要获得某个表中的S lock时，必须先获得这张表的IS lock 或者 IX lock；当一个transaction需要获得某个表中的X lock时，必须先获得这张表的IX lock。

实际上，IS lock 和IX lock 并没有锁表：当一个transaction已经获得了一张表的IX lock或IS lock时，另外一个transaction可以获得这张表的IX lock或者IS lock中的任意一个。Intention lock只起到一个标示作用：某个transaction在某张表中已经获得了某些记录对应类型的row-level lock，或者将要获得这张表中某些记录对应类型的row-level lock。此外，当transaction获得了intention lock时，该transaction访问表中的记录时，会加上与intention lock同类型的row-level lock。

S lock 和X lock之间的互斥性是显而易见的，而IS lock 和 IX lock 之间没有互斥性。row-level lock和intention lock之间是没有互斥性的，因为属于不同的level。至于官方文档里面那张奇怪的表格，貌似是没表述清楚，具体解释可以[参照这个][description1]。


# 获取intention lock

如果只是执行select ... 语句不会获得任何锁，也会忽略所有锁，因为InnoDB默认状态下使用的是consistent reads。

select ... lock in share mode会获得IS lock；select ... for update 会获得IX lock。

update、insert 和 delete都会获得IX lock。

# 获取row-level lock

总体来讲，InnoDB是这样来获得row-level lock的：在搜索表的时候，对任何扫描过的行，都会获得这些行的锁。换句话说，如果因为没有建索引而进行了全表搜索，则所有行都会被锁住；如果建了索引，只有在搜索索引过程中命中了的行才会被锁住。

row-level lock可以理解成read-write lock，但是具体来讲，row-level又分为三种类型：record lock, gap lock 和 next-key lock。

record lock仅仅是锁住一条或多条记录。

gap lock可以锁住某一个gap。所谓的gap，就是index中值与值之间的空隙。举例而言，在一个整数类型的列上建了索应，则(1,20)可以是一个gap。如果这个gap被加上了锁，那么(1,20)之间的row无法被插入。除了两个值之间，类似于(负无穷,1)和(20,正无穷)也可以是一个gap。

next-key lock就是一个record lock加上一个gap lock，其中gap lock加在对应record之前的一个gap上。比如说，一个index上有10、11、13、20这么几个值，next-key lock可以是：(负无穷,10]，(10,11]，(11,13]，(13, 20]，(20,正无穷)。

在官方文档上暂时没有找到明确的加锁规则，因此先列举一些样例吧：

1. 如果在index上有值10、 20，那么select ... where id > 15 for update 会锁住20以及(10,正无穷)，select ... where id < 15 for update 会锁住10以及(负无穷,20)。

2. 如果在index上有值10、 20，那么select ... where id in (10, 13, 20) for update 会锁住10、 20以及(10,20)，select ... where id in (15, 20)会锁住20以及(10,20)，select ... where id in (13, 15)会锁住(10,20)。

# 避免死锁

如果需要对单张表的多条记录进行修改，则在transaction一开始就应该获取所有的锁。

如果无法一次性获得单张表的所有锁，或者需要对多张表进行修改，则按全局一致的顺序依次获得所有锁。

MySQL自己会对死锁进行检测，在必要的时候强制中断transaction。

show engine innodb status 命令可以查看transaction中锁的情况，设置innodb_status_output_locks = 1时可以看到lock的详细情况。Mariadb在10.0.14版本才加入了innodb_status_output_locks配置项。

# 应用举例

1. 对于两张表parent和child，如果我们select出一条parent判断其是存在的，然后若需要对其插入一条child记录，那么select需要获得S lock，如果插入child的同时需要修改parent，那么select需要获得X lock。


[description1]: http://bugs.mysql.com/bug.php?id=63665
