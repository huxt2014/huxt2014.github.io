---
layout: post
title:  "Python中的那些开源ORM"
date:   2017-02-19 17:48:08 +0800
categories: python
tags: web
---

ORM(object relationship mapping)解决的是在使用面向对象的编程风格访问关系型数据库时，对象的设计与表的设计不匹配问题。在Java中，耳熟能详的就是Hibernate了，在Python中比较有名的是SQLAlchemy与Django ORM，此外还有其他一些迷你的ORM。

这篇文章从设计、mapping、灵活性、易用性、外部集成等方面归纳一下Python中各个ORM的不同之处，在面临选择时以供参考。

---

# SQLAlchemy

最近两年一直在使用SQLAlchemy，这也是唯一一个深入使用过的ORM。首先向SQLAlchemy的社区表达一下感谢吧，感谢社区提供的开源代码以及维护了如此详细、完善的文档。

**设计**

从总体上来讲，SQLAlchemy分为两个部分，Core和ORM。Core部分提供了基础但是比较全面的功能，包括expression query API，作为layer屏蔽DBAPI的差异，定义、反射、操作数据库的schema等等。而ORM建立在Core基础之上，提供了mapping的功能。可以说，前者负责的是Python-SQL mapping，后者负责的是object-relationship mapping.

虽然被拆分为Core与ORM两部分，但是两者在使用上是高度兼容的，两者提供的接口也高度统一，例如用Core写的query可以直接放到ORM中执行。不仅仅是Core与ORM之间接口统一，SQLAlchemy整体上都遵循了统一的设计风格。

从设计模式上看，SQLAlchemy实现了data mapper, unit of work，identity map等[Patterns of Enterprise Application Architecture][PEAA]中提到的大多数相关的模式。SQLAlchemy的作者表示用了很多年才把这些特性搞定，并且积累了茫茫多的测试用例。

**mapping**

对象设计和表设计的差异性主要体现在关系和继承，这也是ORM的基本功能。

关系方面，多对多、一对多、多对一这样基础的支持自然不在话下，adjacency list pattern也是OK的。此外，还可以配置两张表之间如何join。其他更多的特性可以参考文档。

继承方面，SQLAlchemy支持single table inheritance、concrete table inheritance和 joined table inheritance。

**灵活性**

由于Core和ORM的独立性和兼容性，使用SQLAlchemy是相当灵活的。你可以单独使用Core，或者单独使用ORM，或者杂糅在一起用。

SQLAlchemy是一个独立的工具，不与其它东西绑定。你可以用它进行web开发，也可以用来做ETL工具，可以做任何与数据库交互的东西。

**易用性**

SQLAlchemy的学习曲线有点平缓，上手相比其他ORM比较慢。但是换个角度而言，慢的原因主要在unit of work、identity map以及其他特性的熟悉，并不是SQLAlchemy自身比较难用。好在官方文档详细且完整，组织得也恰到好处。

SQLAlchemy没有附带migration的工具，需要借助其他的工具来完成。SQLAlchemy的作者开发了alembic，而Openstack使用的是sqlalchemy_migrate

**外部集成**

由于SQLAlchemy是一个独立的工具，所以在特定的领域使用时，可能需要做一些额外的工作。比如，当用于web开发时，与对应的web框架集成需要一些额外的工作量。好在使用的人多了后，社区的支持也会随之跟上，使用Flask作为web框架时，就有Flask-SQLAlchem提供对应的支持。

不过也有比较好的例子，例如pandas访问数据库默认支持的SQLAlchemy，yosai（向shiro看齐的一个年轻的项目）也默认使用的是SQLAlchemy。

若干年前作者表示，SQLAlchemy的最大缺点可能就是独立于Django生态链了。但是随着wsig的标准发布，Flask、Gevent、Gunicom等web领域工具的出现，以及非Web领域的工具选择SQLAlchemy作为默认的数据库引擎，这应该不算是遗憾了吧。

**其他**

社区一直在维护和更新SQLAlchemy，文档非常完善，且OverStackflow上的回答也不少。从2006年发布迭代至今，性能和稳定性都不断在提升，至少经受住了时间考验。Peewee的作者评价SQLAlchemy为the gold standard for ORM in the Python world.

---

# Django ORM

Django为python的web开发者提供了一揽子解决方案，其中也包含了访问数据库的ORM，即Django ORM。按照Django文档中的介绍，当时Django启动的时候，并没有找到合适的ORM，于是就自己写了。

**设计**

Django ORM 使用的是active record，至于unit of work, identity map，从文档上看感觉是没有的。

Django ORM貌似只专注于OOP，并且只专注于Django本身，所以很多功能给人的感觉是够用就行。SQLAlchemy作者吐槽说Django ORM不够贴合SQL，对不同数据库的支持实现得也比较晚，不像SQLAlchemy那样专注于数据库。

Peewee作者吐槽Django ORM提供的接口不够简洁和统一，例如同样一个查询条件有多种不那么直观的写法（这让我想起了Perl）。之所以会这样，原因很大程度上可以归咎于Django ORM缺少完整独立的expression query API。

**mapping**

关系方面，基础的几种都支持，但是从文档上没有找到高级配置的说明。

继承方面，虽然官方文档上说有三种风格，但实际上只支持了joined table inheritance（官方文档的说法是multi-table inheritance)。

**灵活性**

Django ORM属于Django的一部分，所以缺少独立性，“我想写一个简单的命令行工具访问数据库所以我安装了Django”这样的逻辑感觉也是蛮不直观的。至于Django有没有提供把ORM单独拎出来的方法，还需要再核实一下。

Django ORM没有一个独立的layer来完成Python到SQL的映射。

**易用性**

active record加上Django为web开发提供的一揽子解决方案，决定了Django ORM可以很快上手。Django自带的migrate工具也是相当赞的。其实，说到头还是Django的易用性。

**外部集成**

就不用想外部了，只考虑Django就够了。其他第三方工具，如果不是针对Django的，估计也不会使用Django ORM来支持自己访问数据库。

---

# Peewee

SQLAlchemy提供了data mapper，Django ORM提供了active record，按理说就没其他工具什么事了。但是，由于Django ORM一些显著的特点，所以给了其他工具一些机会，Peewee就是其中之一。

**设计**

Peewee采用了active record模式，其最大特点就是小巧。它的作者对其优缺点作了一些[阐述][peewee]，可以看到Peewee的特点造就了Peewee的优点与缺点。

由于Peewee作者吐槽了Django ORM的接口风格不一致问题，Peewee在这一块应该是有针对性改进的。此外，Peewee在expression方面做了一些改进，有SQLAlchemy的身影。

粗略地看了一下文档，感觉上Peewee在active record上沿袭了Django ORM的风格，但是又加入了SQLAlchemy中的expression query API的风格。由于Peewee的目标是小巧，感觉应该不会像SQLAlchemy那样独立出一个Core部分。

**其他**

由于Peewee比较年轻，因此社区支持较薄弱，目前貌似还不支持Oracle和Microsoft SQL Server。但是，如果真的在功能上能够向Django ORM看齐，弥补一些缺点后作为一个独立的ORM工具还是蛮不错的。

---

# MongoEngine

MongoEngine是Document-Object Mapper，基本上是与Mongo绑定了。


[PEAA]: https://www.amazon.cn/dp/0321127420/ref=sr_1_2?ie=UTF8&qid=1487410272&sr=8-2&keywords=Patterns+of+Enterprise+Application+Architecture
[peewee]: https://www.reddit.com/r/Python/comments/4tnqai/choosing_a_python_ormpeewee_vs_sqlalchemy/
