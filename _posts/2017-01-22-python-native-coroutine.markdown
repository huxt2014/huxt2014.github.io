---
layout: post
title:  "Python基础：native coroutine的前世今生"
date:   2017-01-22 14:48:08 +0800
categories: python
tags: python
---

继gevent之后，又慢慢在填python native coroutine的坑。虽说属于CPython官方原生支持，但是为啥native coroutine感觉没有gevent那么直观呢。不过好歹也算是慢慢弄懂了，用帖子来记录一下对native coroutine的理解。

native coroutine以及await、async def是由[PEP 492][PEP0492]提出并确认的。这个PEP在2015年才被创建，感觉是属于比较新的内容，新出现的东西网上资料可能相对难找，所以就从PEP啃起吧。PEP 492中提到希望读者已经了解了[PEP 342][PEP0342]和[PEP 380][PEP0380]，尼玛这是买一赠二啊。虽然不太情愿，但是为了保证知识的连贯性，也只有从头开始了。

# 遥远的年代

很久很久以前，当Python中还只有iteration protocol和generator的时候，故事远远没有现在这么复杂。generator让我们可以利用yield，通过写函数来快速地实现iterable。是的，就像如下代码那样简单。

{% highlight ipython %}

In [25]: def gen():
    ...:     i = 10
    ...:     while i >= 0:
    ...:         yield i
    ...:         i -= 1
    ...:         

In [26]: list(i for i in gen())
Out[26]: [10, 9, 8, 7, 6, 5, 4, 3, 2, 1, 0]

{% endhighlight %}

这个时候，yield是一个statement，在iteration context调用__next__时返回一个对象、挂起generaotr转而执行caller的逻辑。当iteration context再次调用__next__时，被挂起的generator会重新执行，达到一个重入的效果。

若干次重入和挂起之后，如果再次重入的时候，generator的函数体执行完了/像函数一样返回了/执行不到下一个yield了（这三个说法是一个意思），那么就会抛出一个StopIteration。Iteration context会捕获这个异常，然后知道迭代完了，可以执行后面的代码了。捕获StopIteration时，iteration context只知道不会再有新的东西被yield出来了，其并不关心generator有没有提供额外的信息。

# 从statement到expression

2005年[PEP 342][PEP0342]被创建的时候，已经有coroutine的概念了。这个PEP中将yield由statement转变为了expression，并且为generator增加了send、throw、close等方法。增加这些内容的主要目的，就是让caller可以向generator里面传递对象，而不仅仅是generator向caller传递对象。

Python中，任何expression都可以计算得到一个对象，那么yield作为一个expression会得到什么呢，那就是send传入的值。send与next函数类似，可以让挂起的generator重新执行。利用send让generator重新执行之后，yield expression的值就是send传入的对象，然后依次执行后续的逻辑，直到下一个yield向caller返回一个值后挂起。

{% highlight ipython %}
In [27]: def gen():
    ...:     i = 0
    ...:     while True:
    ...:         value = yield i
    ...:         print('get value: ',value)
    ...:         

In [28]: g = gen()

In [29]: next(g)
Out[29]: 0

In [30]: r = g.send('hello')
get value:  hello

In [31]: r
Out[31]: 0

{% endhighlight %}

在这个年代，StopIteration依旧是起一个提示作用，caller并不关心generator结束时候的状态。

# yield from和subgenerator

2009年，[PEP 380][PEP0380]被创建，在这个PEP中，有了subgenerator的概念以及yield from（这是一个expression），并且generator在结束时可以向caller返回对象了。

在此之前，如果一个generator里面需要使用另外一个generator（下文称之为subgenerator），那么使用subgenerator的方法只用两种：将其返回或将其展开。有了yield from后，一方面情况变得简单了，因为subgenerator可以直接向外面传值；另一方面send的功能更强了，因为send可以向subgenerator中传值。如下示例可以表示三种方式的差别。

{% highlight python %}

def sub():
    for i in range(5):
        yield i

def gen1():
    yield sub()          # 整体返回

g = gen1()
g.next()                  # caller拿到的是sub()

def gen2():
    for v in sub():
        yield v           # 展开后向caller yield对象
        
g = gen2()
g.next()                  #caller拿到的是v

def gen3():
    yield from sub()     # sub中的yield可以直接向gen3的caller返回对象
    
g = gen3()
g.next()
g.send(None)             # caller可以直接传值给sub中的yield expression
    
{% endhighlight %}

值得注意的是，subgenerator里面依旧可以使用yield from，多次嵌套后就形成了一个yield from的链条，该链条的最底层就是一个yield expression了。caller调用next和send的时候，是和最底层的yield expression通信。

此外，generator里面可以有return了，return的值会附带在StopIteration上。yield from 作为expression，它的值就是subgenerator中return的值，也就是StopIteration上附带的值。具体使用方法见如下示例。

{% highlight ipython%}

In [6]: def sub():
   ...:     yield 'yield from sub'
   ...:     return 'return from sub'
   ...: 

In [7]: def gen():
   ...:     value = yield from sub()
   ...:     print('gen get from sub: %s' % value)
   ...:     yield 'yield from gen'
   ...:     return 'return from gen'
   ...: 

In [8]: g = gen()

In [9]: next(g)
Out[9]: 'yield from sub'

In [10]: next(g)
gen get from sub: return from sub
Out[10]: 'yield from gen'

In [11]: next(g)
StopIteration: return from gen

{% endhighlight %}

# native coroutine

从generator的演变历程上可以看到，Python利用generator来达到了coroutine的效果，但是这容易将generator与coroutine搞混淆。为了修正这个问题，[PEP 492][PEP0492]被创建出来。这个PEP中提出了新的关键字async def 和await，利用这两个关键字将coroutine与generator分割开来。

在此之前，如果一个函数定义def里面出现了yield，那么这个函数会被当成generator，而generator可以达到corontine的作用。现在，coroutine由async def定义，函数体中不可以出现yield、yield from，但是可以出现return和await。

await与yield from一样，是一个expression。yield from后面接的是generator，而await后面接的是与coroutine协议兼容的对象，具体可以接哪几种对象可以参考PEP。coroutine与generator类似，完成后都会返回一个值，也是通过StopIteration带出来的，而这个值就是await expression的值啦。示例如下所示。

{% highlight ipython%}
In [18]: async def sub():
    ...:     return 'return from sub'
    ...: 

In [19]: async def co():
    ...:     value = await sub()
    ...:     print('get value from sub: %s' % value)
    ...:     return 'return from co'
    ...: 

In [20]: c = co()

In [21]: c.send(None)
get value from sub: return from sub
StopIteration: return from co

{% endhighlight %}

一开始对于await这个关键字我是拒绝的，因为感觉很奇怪，找不到已有知识与之形成对照，直到某一天我想起了进程中的wait。在多进程中，wait函数等待另外一个进程的完成，并且获得该进程的termination status。如果这样想的话，await确实是等待另外一个corontine的完成，并且获得其返回值。

到此为止，已经把native coroutine的发展历程大概回顾一下。以上提到的PEP里面还有一部分内容没介绍到，更详细的信息直接阅读PEP的原文吧。

最后，这片文章里面并没有介绍coroutine是如何并发的，这个就涉及到aysncio包了，下篇文章再细说。

[PEP0492]: https://www.python.org/dev/peps/pep-0492/
[PEP0342]: https://www.python.org/dev/peps/pep-0342/
[PEP0380]: https://www.python.org/dev/peps/pep-0380/

