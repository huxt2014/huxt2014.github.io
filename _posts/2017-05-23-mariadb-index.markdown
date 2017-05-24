---
layout: post
title:  "Mariadb中索引的基本使用"
date:   2017-05-23 19:48:08 +0800
categories: mariadb
---

这篇文章记录了mariadb中索引的基本概念以及一般性用法，内容绝大部分来自于对mariadb官方文档、mysql官方文档以及各类博客的的归纳与总结，方便实际使用中的快速查阅。对于特定的使用场景，还需参考官方文档或咨询专家。


# 基本原理

---

#### 类别

mariadb中的索引主要分为四类：primary keys，unique indexes，plain indexes，full-text indexes。

**primary key：** 每张表有且只有一个，不允许有NULL。如果没有指定且不存在UNIQUE index，XtraDB/InnoDB 会创建一个（用户不可见）。

primary key是与数据存储在一起的，“clustered”在BTree上，因此XtraDB/InnoDB可以根据主键来调整数据的存储，这使得（按index）顺序读会比较快。在数据量很大时，可能有多个level的BTree。其他类型的索引是单独存储的，primary key存储在叶子节点上（如果用到BTree的话）。

**unique index：** 允许存在NULL，且允许存在多条NULL的记录。

**plain index：** 普通的索引，可以定义在一列或者多列上。如果超过长度（默认配置为767），只会选取左边的一部分（prefix index）。XtraDB/InnoDB中，plain index会带有主键信息。

**full-text index：**

**spital index：** 用于多维数据检索，具体信息待确认。

#### 实现

**BTree：** 可以对WHERE语句中的等式、不等式、范围选择起到优化作用。XtraDB/InnoDB中默认用的是B-tree。

**Hash Table：** 仅对等式和不等式（!=）起到优化左右，对范围选择、group by、order by等无效。不可以使用prefix index，具有接近于O(1)时间复杂度。

# 选择索引

---

#### primary key

选择primary key的时候，所选定的列需要满足如下条件。

**简单：** 包括数据类型和列的数量。在满足需求的情况下，整数优于字符串，单列优于多列。主键出现在查询条件中的概率比较大，并且常被当作外键用于join，简单的数据类型做比较效率更高，特别是在记录条数比较多的时候。此外，primary key会被存储在其他索引中，简单一点可以节省资源。

**不会改变：** 记录一单生成，其主键的值不能发生改变。在关系型数据库中，某条记录主键一旦需要改变，其他表中依赖于这条记录的内容都需要更改，这会造成各种不必要的麻烦。

**有限的含义：** 主键中的值不应该带有有多余的业务范畴的含义，这是因为业务上的变化往往要求数据库中对应列的值也发生变化，而这样的变化是主键应该尽量避免的。有些业务相关的字段天然地带有主键的特性，例如记录某台设备上每五分钟某状态变化的表，那么设备id、时间戳、状态量id就可以作为主键，而没必要另外构建一个无业务属性的列作为主键。

**低随机性：** 数据库会根据主键的值来调整数据在磁盘上的存储顺序，主键的值如果随机性比较高，可能会造成存储比较分散，带来额外的磁盘IO。例如，避免主键中存储的是哈希值。

如果有多个备选项，则更倾向于对关键查询影响更大的选项。


#### plain key

对于在哪些列上建立索引能够优化查询，mariadb官方文档给出一个简单通用的算法：

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

从以上算法可以推断，XtraDB/InnoDB在使用索引进行优化时是如何选择索引的。当索引列出现在等式中时，会优先使用；等式中的索引列和其他表达式（不等式、GROUP BY、ORDER BY）中的索引列可以协同工作；一般来讲不等式、GROUP BY和ORDER BY中的索引列无法协同工作，使用的优先顺序是不等式 > GROUP BY > ORDER BY。

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
但是，这个对于主键列不起效果的，因为主键列与数据存在一起。对于被截取的index（prefix index）
也是没有效果的。有效示例如下：
    SELECT x,z FROM t WHERE y = 5 AND q = 7     INDEX(y,q,x,z)
    SELECT x FROM t WHERE y > 5 AND q > 7       INDEX(y,q,x)
3. INDEX(a, b)的检索作用可以覆盖INDEX(a)的检索作用，所以可以去除多余的索引。
4. 有时候优化器会跳过WHERE中的索引而直接使用ORDER BY中的索引，前提是索引与ORDER BY中的列完
全匹配。当查询中存在LIMIT是，这样做效果会更好。
5. 如果使用了prefix index，效果一般不怎么好。
6. 对于IN的优化要看mariadb／MySQL的版本，较新的版本才提供了查询优化。
7. 索引会带来一些负面效果：占用存储空间，过多索引让优化器作选择时更加耗时，增、删、改的过程更
加耗时（需要修改索引）。
8. 使用EXPLAIN可以查看执行计划。

{% endhighlight %}

#### 其他

查询中会查询不同组合的列，有些组合查询频率比较低，那么可以把查询频率低的列单独分出一张表。

对于INDEX(c1, c2)，在条件WHERE c2='v' and c1 between 'v1' and 'v2'上无法起到优化作用。


[案例1][case1]

[案例2][case2]

# 查询优化器

---

建立索引的目的是加快查询的速度，这个目标是通过查询优化器（query optimizer）来达成的。EXPLAIN可以用来查看查询优化器如何利用索引进行优化的。

#### 全表扫描

如果optimizer无法找到可以使用的索引，那么会进行全表扫描，如下所示。

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

#### 单个索引

建立索引Index(name)后，optimizer就可以利用索引了：

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

#### 多个索引

如果optimizer找到多个可以使用的索引，那么可以讲两个索引一起使用。大概原理是针对每个索引都可以找到primary key的集合，然后取交集就可以了。建立Index(Name) 和 Index(CountryCode)，有如下效果：

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

#### 自定义

如果不满意optimizer使用的索引，那么可以为optimizer指定索引。在已知数据分布的情况下，这样可以减少optimizer的工作。

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

[case1]: https://stackoverflow.com/questions/28974572/mysql-index-for-order-by-with-date-range-in-where
[case2]: https://stackoverflow.com/questions/37022941/mysql-optimization-for-large-myisam-table/37024870#37024870