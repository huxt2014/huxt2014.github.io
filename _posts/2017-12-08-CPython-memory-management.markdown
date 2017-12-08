---
layout: post
title:  "CPython：内存管理"
date:   2017-12-08 21:00:00 +0800
categories: python
tags: python
---

为了避免频繁地使用系统调用申请和释放小内存，CPython有自己的针对小内存的内存管理系统，这篇文章简要介绍了这个内存管理系统的实现原理。

# 概述

---

CPython在运行过程中需要频繁地申请和释放位于heap上的小内存，所谓的小内存指的是几十k到几百k的内存。这些申请和释放操作如果仅仅依赖系统调用，频繁地操作会造成运行速度下降。因此，CPython的内存管理系统维护了一个内存池，当需要申请小内存时，会从该内存中分配，而非使用系统调用。

CPython的内存管理系统的层次逻辑如下所示：

{% highlight text %}
    Object-specific allocators
    _____   ______   ______       ________
   [ int ] [ dict ] [ list ] ... [ string ]       Python core         |
+3 | <----- Object-specific memory -----> | <-- Non-object memory --> |
    _______________________________       |                           |
   [   Python's object allocator   ]      |                           |
+2 | ####### Object memory ####### | <------ Internal buffers ------> |
    ______________________________________________________________    |
   [          Python's raw memory allocator (PyMem_ API)          ]   |
+1 | <----- Python memory (under PyMem manager's control) ------> |   |
    __________________________________________________________________
   [    Underlying general-purpose allocator (ex: C library malloc)   ]
 0 | <------ Virtual memory allocated for the python process -------> |

{% endhighlight %}

第1层的raw memory allocator只是对C library malloc的简单封装；第2层的object allocator管理了一个内存池，主要管理小于512K（默认值）内存的申请和释放；第3层的Object-specific allocators针对几种内置的数据类型分别实现了不同的策略。这篇文章主要介绍第2层和第3层的实现原理。


# object allocator

---

CPython为特定大小的内存块维护了特定的pool，例如1~8 bytes的内存块只会在若干个pool上进行分配，9～16 bytes的内存块只会在另外若干个pool上进行分配，1～8 bytes的内存和8～16 bytes的内存属于不同的class。哪个class对应了哪几个pool由一个称为usedpools的数组维护，示意图如下：

{% highlight text %}
                    
usedpools[0+0] --> +-------------+
                   |             |
                   +-------------+
                   |             |
usedpools[1+1] --> +-------------+ <-- usedpools <-----+
                   |  8-nextpool |                     |
                   +-------------+                     |
                   |  8-prevpool |                     |
usedpools[2+2] --> +-------------+     +--+     +--+   |
                   | 16-nextpool | <-->|p0| <-->|p1| --+  
                   +-------------+     +--+     +--+
                   | 16-prevpool | ------------->|
    ...            +-------------+
                   |     ...     |
                   +-------------+
                   |     ...     |
 
{% endhighlight %}

如果要完全理解，需要参考源码中usedpools的定义。简单来讲，属于某个class的pool由双向链表串起来，而双向链表的头部记录在usedpools中。

每一个pool的大小是4K（默认值），也就是一个page大小。如果是32 bytes的object，一个pool上可以分配100多个object。这些object在pool上分配和释放的示意图如下所示：

{% highlight text %}

                                    used     used   NULL as sentinel
     <      pool header         >     |        |     |
     +---+---------+----+----+---+--------+--------+----+----+--------+
     |ref|freeblock|next|prev|...|32 bytes|32 bytes|NULL|... |32 bytes|
     +---+---------+----+----+---+--------+--------+----+----+--------+
             |                                                    |
             +------------------------------------>|              |
                                                            the last segment
                                                           will never be used

                   |                                  /|\
  free one segment |                                   | use one segment
                  \|/                                  |

                                    free    used
                                      |       |
     +---+---------+----+----+---+--------+--------+----+----+--------+
     |ref|freeblock|next|prev|...|32 bytes|32 bytes|NULL|... |32 bytes|
     +---+---------+----+----+---+--------+--------+----+----+--------+
             |                     |
             |                     +-------------->|
             +------------------>|


{% endhighlight %}

freeblock指向了下一个可用的segment的位置，同时也是链表的头指针，维护了一个链表。当某个segment被释放的时候，会添加到这个链表的头部。所以，这个是一个生存在数组上的链表。

当usedpools中空间不够的时候，就会申请一个新的pool，当一个使用中的pool全部空出来后，这个pool会被释放。pool的申请和释放并不是直接使用系统调用，而是由arena管理。

arena是一个256K（默认值）的内存块，一个arena可以管理64个pool，所有的arena有链表串起来，其示意图如下：

{% highlight text %}

     |    ...     |   +--usable_arenas
     |    ...     |   |
     +------------+ <-+   +--------------+
     |  address   | ----> |p0|p1|p2|...  |   256K bytes block = 64 pages/pools
     +------------+       +--------------+
     |pool_address| -------------->|         next pool that never used
     +------------+
     | nfreepools |
     +------------+
     | ntotalpools|
     +------------+    +--+    +--+
     |  freepools | -->|p1| -->|p2|          have used sometime but now freed
     +------------+    +--+    +--+
     | nextarena  |
     +------------+
     | prevarena  |
     +------------+ <- ununsed_arena_objects
     |    ...     |
     |    ...     |

{% endhighlight %}

freepools维护了一个链表，当一个pool被释放后会被添加到这个链表的头部，usedpools申请新的pool时也会优先从这里拿。同样，这也是一个生存在数组上的链表。

# Object-specific allocators

----

对于内置类型的内存管理，主要就是每个类型都有自己的缓存区。在一定范围内，缓存区内的对象会重复使用或者共享。

像list、dict这类容器类型，缓存区就是一个数组，以dict为例，如下所示：

{% highlight C %}

static PyDictObject *free_list[PyDict_MAXFREELIST];
static int numfree = 0;

static PyObject *
new_dict(PyDictKeysObject *keys, PyObject **values)
{
    PyDictObject *mp;
    assert(keys != NULL);
    if (numfree) {
        mp = free_list[--numfree];
        ...
    }
    ...
}

static void
dict_dealloc(PyDictObject *mp)
{
    ...
    if (numfree < PyDict_MAXFREELIST && Py_TYPE(mp) == &PyDict_Type)
        free_list[numfree++] = mp;
    else
        Py_TYPE(mp)->tp_free((PyObject *)mp);
    ...
}

{% endhighlight %}

像int、unicode这样的不可更改的类型，部分对象是共享的。以int为例，小整数是共享的，如下所示：

{% highlight C %}

static PyLongObject small_ints[NSMALLNEGINTS + NSMALLPOSINTS];

#define CHECK_SMALL_INT(ival) \
    do if (-NSMALLNEGINTS <= ival && ival < NSMALLPOSINTS) { \
        return get_small_int((sdigit)ival); \
    } while(0)

static PyObject *
get_small_int(sdigit ival)
{
    PyObject *v;
    assert(-NSMALLNEGINTS <= ival && ival < NSMALLPOSINTS);
    v = (PyObject *)&small_ints[ival + NSMALLNEGINTS];
    ...
}

PyObject *
PyLong_FromLong(long ival)
{
    ...
    CHECK_SMALL_INT(ival);
    ...
}

{% endhighlight %}
(the end)
