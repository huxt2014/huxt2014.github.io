---
layout: post
title:  "MySQL/Mariadb：索引的基本使用"
date:   2017-05-23 19:48:08 +0800
categories: mariadb
tags: web
---

这篇文章记录了Mariadb（主要是InnoDB）中索引的基本概念以及一般性用法，内容绝大部分来自于对Mariadb官方文档、MySQL官方文档以及各类博客的的归纳与总结，方便实际使用中的快速查阅。对于特定的使用场景，还需参考官方文档或咨询专家。


# 类别

---

按基本功能来划分，mariadb中的索引可以分为五类：primary key，unique index，full-text index，spatial index，plain index：

1. primary key，也就是主键，不允许有NULL。

2. unique index，要求所有key都是不重复的。允许有NULL，并且允许多条记录的key为NULL。

3. full-text index，可以用于full-text search，允许有NULL。

4. spatial index，用于多维数据检索，不允许有NULL。

5. plain index，也就是普通的索引，允许有NULL。

此外，如果在多个列上建立一个索引，那么可以称为multiple-column index或composite index，例如INDEX(a, b, c)，key的值由对应的列拼接而得到。可以在某一列的前若干长度上建立索引，这种索引称为prefix index。

# 索引的结构

---

InnoDB中索引的实现方式都是B+tree，即使在创建索引的时候指定为hash table，最后得到的还是B+tree。可以说，InnoDB中一张表的所有内容都以索引的方式存在：1）所有的记录都存在clusterd index中，这颗B+tree的key就是某一行记录的主键，value就是某一行记录的数据。2）其他的索引称为secondary index，他们拥有相同的结构，key就是某一行中用来建立索引的那一列，value就是这一行对应的主键。

clusterd index是一个特殊的索引，通常情况下就是主键。如果创建表的时候没有指定主键，那么InnoDB会选择第一个不允许有NULL的unique index作为clusterd index。再不然，InnoDB会自动生成一个用户不可见的clusterd index。

假设有一张表，其主键id是int类型，对应的数据是data，那么有8条记录（id=1～8）的情况下，clustered index的示意图大致如下：

{% highlight text %}

                           +--------------------+
                           |       page 3       |
                           | +-----+    +-----+ |
                           | | >=0 | -> | >=4 | |
                           | | ->4 |    | ->5 | |
                           | +--|--+    +--|--+ |
                           +----|----------|----+
                        +-------+          +-------+
             +----------|---------+     +----------|---------+
             |        page 4      |     |        page 5      |
             | +-----+    +-----+ |     | +-----+    +-----+ |
             | | >=0 | -> | >=2 | |     | | >=4 | -> | >=6 | |   internal page
             | | ->6 |    | ->7 | |     | | ->8 |    | ->9 | |
             | +--|--+    +--|--+ |     | +--|--+    +--|--+ |
             +----|----------|----+     +----|----------|----+
            +-----+          +--------+
 +----------|---------+    +----------|---------+
 |        page 6      |    |        page 7      |
 | +-----+    +-----+ |    | +-----+    +-----+ |
 | |  0  | -> |  1  | |    | |  2  | -> |  3  | |       ...      leaf page
 | | data|    | data| |    | | data|    | data| |
 | +-----+    +-----+ |    | +-----+    +-----+ |
 +--------------------+    +--------------------+


{% endhighlight %}

默认情况下每个page的大小是16KB，主要的数据存储在leaf page中，每个page的实际使用量大概在1/2～15/16之间。当然，上图仅仅是示意图，每个page中的数据不可能这么少，使用B+tree的原因就是让每个page存尽量多的key，然后尽可能降低B+tree的高度。

B+tree与hash table有不同的特点：B+tree可以对WHERE语句中的等式、不等式、范围选择起到优化作用，对非%、_开头的like也有作用。hash table仅对等式和不等式（!=）起到优化左右，对范围选择、group by、order by等无效，无法估计某一个范围内大概有多少行，但是查询时具有接近于O(1)时间复杂度。

# 选择索引

---

选择**主键**的时候，所选择的列需要尽量满足如下条件。

1. 简单，包括数据类型和列的数量。在满足需求的情况下，整数优于字符串，单列优于多列。主键出现在查询条件中的概率比较大，并且常被当作外键用于join，简单的数据类型做比较效率更高。此外，主键会被存储在其他索引中，简单一点可以节省资源。

2. 不会改变，即记录一单生成，其主键的值不会发生改变。在关系型数据库中，某条记录主键一旦需要改变，其他表中依赖于这条记录的内容都需要更改，这会造成各种不必要的麻烦。

3. 有限的含义，即尽量避免带有业务属性的列。如果主键含有过多的业务属性，那么业务发生改变的时候要求主键也随之变化的可能性会更高，而这种对数据库的改变是应该尽量避免的。如果系统中记录的数据是偏向底层的数据，也就是业务变更的可能性不大，那么也可以适当考虑一些带有实际含义的列。另外一方面，由于主键以及外键自带的一些约束性，使用带有业务属性的列当作主键对保证数据的完整性会有帮助。这里面的权衡和取舍还要视具体情况而定。

4. 低随机性。数据库会根据主键的值来调整数据在磁盘上的存储顺序，主键的值如果随机性比较高，可能会造成存储比较分散，带来额外的磁盘IO。随机性的一个极端是哈希值，而非随机性的一个极端是自增列。

5. 选择使用频率更高的列。如果有多个备选项，则更倾向于对关键查询影响更大的选项。如果确实没有合适的column，那么可以考虑使用一个自增列当主键，因为自增列满足以上所有条件。


**普通索引**主要用于加快查询速度，对于数据库结构的影响没有主键那么大，所以要求相对少一些。为了使加速的效果尽可能的好，需要对查询优化器使用索引的机制比较熟悉。对于在哪些列上建立索引能够优化查询，mariadb官方文档给出一个简单通用的算法，这个算法一方面可以为建立索引提供参考，另一方面也可以推测一下查询优化器的工作机制。算法如下：

{% highlight text %}

给定一条SELECT语句，按如下步骤选出列，将列按选出的顺序排列，即可得到一个不错的索引：
  1. 将WHERE语句中用AND连接的“column=const”、“column is NULL”表达式挑出来，选择对应的
列。
  2. 在剩余的表达式中，按如下匹配到的第一条规则选出列：
    2a. 存在范围选择表达式，例如>,BETWEEN，IS NOT NULL，LIKE without leading
wildcard，不能是!=。若存在多个表达式，选择一个。
    2b. 存在GROUY BY语句，选择GROUP BY中的所有列。
    2c. 存在ORDER BY语句且没有用到ASC或DESC，选择ORDER BY中的所有列。

以上，列与常量都因该是直接比较，不能作为函数的参数。如果SELECT是多表查询，不需要考虑其他表。
{% endhighlight %}

以下是一些例子：

{% highlight text %}
WHERE aaa = 123 AND bbb = 456 AND ...       INDEX(aaa,bbb) 或 INDEX(bbb,aaa)
WHERE aaa >= 123 AND bbb = 1                INDEX(bbb, aaa)
WHERE aaa >= 123 AND ccc > 'xyz'            INDEX(aaa) 或 INDEX(ccc)
WHERE aaa >= 123 GROUP BY xxx               INDEX(aaa)
{% endhighlight %}

使用索引时，有一些特殊情况／技巧是需要注意的：
{% highlight text %}
1. 如果按照索引检索出的记录条数大于总数的20%，那么这个索引对优化查询几乎没有效果，有时甚至会降
   低查询速度（磁盘IO顺序读更快）。
2. 如果查询的列属于索引列，那么查询的时候只需要读取索引就可以了，这样可以达到更快的查询速度。
   但是，这个对于主键列不起效果的，因为主键列与数据存在一起。对于prefix index也是没有效果的。
   有效示例如下：
       SELECT x,z FROM t WHERE y = 5 AND q = 7     INDEX(y,q,x,z)
       SELECT x FROM t WHERE y > 5 AND q > 7       INDEX(y,q,x)
3. INDEX(a, b)的检索作用可以覆盖INDEX(a)的检索作用，但是不能覆盖INDEX(b)的作用。
4. 有时候优化器会跳过WHERE中的索引而直接使用ORDER BY中的索引，前提是索引与ORDER BY中的列完
   全匹配。当查询中存在LIMIT是，这样做效果会更好。
5. 如果使用了prefix index，效果一般不怎么好。
6. 对于IN的优化要看mariadb／MySQL的版本，较新的版本才提供了查询优化。
7. 索引会带来一些负面效果：占用存储空间，过多索引让优化器作选择时更加耗时，增、删、改的过程更
   加耗时（需要修改索引）。
8. 使用EXPLAIN可以查看执行计划。
9. 如果两张表进行连接，且相关的column上创建了索引，那么相同类型的column可以更佳有效地利用索
   引。
10. 查询中会查询不同组合的列，有些组合查询频率比较低，那么可以把查询频率低的列单独分出一张表。
    单独分出的表上与原表使用一样的主键。
11. 对于太长的组合索引，可以考虑用它们的hash值作为索引，前提是hash过后的值比较短且不会重复。
    例如hash_col=MD5(CONCAT(val1,val2))
12. 对于INDEX(c1, c2)，在条件WHERE c2='v' and c1 between 'v1' and 'v2'上无法起到优
    化作用。

{% endhighlight %}

# 查询优化器

---

EXPLAIN可以用来查看查询优化器（query optimizer）如何利用索引进行优化的，下面给出几个例子。

* 全表扫描。如果optimizer无法找到可以使用的索引，那么会进行全表扫描，如下所示。

{% highlight text %}

MariaDB [test]> explain select * from city where countrycode = 'NLD' \G
*************************** 1. row ***************************
           id: 1
  select_type: SIMPLE
        table: city
         type: ALL
possible_keys: NULL
          key: NULL
      key_len: NULL
          ref: NULL
         rows: 4188
        Extra: Using where

{% endhighlight %}

* 单个索引。建立索引Index(name)后，optimizer就可以利用索引了：

{% highlight text %}
MariaDB [test]> explain select * from city where Name = 'Springfield'
    -> and CountryCode = 'USA'\G
*************************** 1. row ***************************
           id: 1
  select_type: SIMPLE
        table: city
         type: ref
possible_keys: ix_name
          key: ix_name
      key_len: 35
          ref: const
         rows: 3
        Extra: Using index condition; Using where
{% endhighlight %}

* 多个索引。如果optimizer找到多个可以使用的索引，那么可以讲两个索引一起使用。大概原理是针对每个索引都可以找到primary key的集合，然后取交集就可以了。建立Index(Name) 和 Index(CountryCode)，有如下效果：

{% highlight text %}

MariaDB [test]> explain select * from city where Name = 'Springfield'
    -> and CountryCode = 'USA' \G
*************************** 1. row ***************************
           id: 1
  select_type: SIMPLE
        table: city
         type: index_merge
possible_keys: ix_name,ix_country_code
          key: ix_name,ix_country_code
      key_len: 35,3
          ref: NULL
         rows: 1
        Extra: Using intersect(ix_name,ix_country_code); Using where

{% endhighlight %}

* 自定义。如果不满意optimizer使用的索引，那么可以为optimizer指定索引。在已知数据分布的情况下，这样可以减少optimizer的工作。

{% highlight text %}
MariaDB [test]> explain select * from city use index (ix_name) 
    -> where Name = 'Springfield' and CountryCode = 'USA'\G
*************************** 1. row ***************************
           id: 1
  select_type: SIMPLE
        table: city
         type: ref
possible_keys: ix_name
          key: ix_name
      key_len: 35
          ref: const
         rows: 3
        Extra: Using index condition; Using where
{% endhighlight %}

# 其他

---

[案例1][case1]

[案例2][case2]


[case1]: https://stackoverflow.com/questions/28974572/mysql-index-for-order-by-with-date-range-in-where
[case2]: https://stackoverflow.com/questions/37022941/mysql-optimization-for-large-myisam-table/37024870#37024870
[ref]: https://blog.jcole.us/2013/01/10/btree-index-structures-in-innodb/