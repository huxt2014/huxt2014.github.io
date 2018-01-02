---
layout: post
title:  "CPython：垃圾回收器"
date:   2017-11-21 21:00:00 +0800
categories: python
tags: python
---

CPython使用引用计数器来完成对象回收和内存释放，这种对象回收机制可以适用于大多数场合，但是有一种场合不适用：对象处于循环引用当中（例如自己引用自己）。针对这种场合，CPython提供了另外一个附加机制，并由gc模块向外暴露了Python层面的API。这片文章主要讲述一下这个附加机制的实现原理。

# 概述

---

传统的垃圾回收机制通常采用如下策略：

1. 找到系统中的root objects
2. 从root objects开始搜寻，通过对象间的引用关系找到所有相关的对象（reachable objects）。
3. 释放unreachable objects。

然而，CPython无法使用上述策略，因为使用C API实现的扩展module让CPython无法准确地找到root set（具体原因待确认）。就算能够找到一种方法能够准确找到root set，由于扩展module可以有各种各样的实现，这种方法很可能是不通用的。因此，CPython在引用计数器的基础之上设计了一个附加的机制，专门用来回收处于循环引用当中的对象。

与传统的策略中先找到reachable objects然后释放unreachable objects不同，CPython使用的方法是直接找到所有unreachable objects。在设计具体的方案之前，有一个特殊的地方需要注意，那就是这个方案是针对可能产生循环引用的对象设计的，而事实上CPython中并不是所有对象都有可能产生循环引用的。所以先要把所有可能产生循环引用的对象单独拎出来，这些对象称为container object。

具体而言，在在创建对象的时候，使用双向列表把container object串起来；此外，除了链表所需的指针字段，再为每个container object准备一个gc_refs字段；对这些串起来的container object按照如下步骤操作：

1. 将每个对象的gc_refs设置为引用计数器相同的数值。
2. 遍历所有对象，对于每个对象，找到其引用的container object，将这些对象的gc_refs减1.
3. 遍历过后，所有gc_refs大于0的container object都被外部引用了，这些对象为reachable objects，将其移到另外一个地方。
4. 在剩余的container object中，如果有被3中移除的container object引用，那么这些也是reachable objects，将其移除。
5. 释放剩余的contaner object。

# container object

---

标记某种类型的对象、某个具体的对象是否为container object的信息记录在PyTypeObject结构体的两个字段当中，分别为tp_flags和tp_is_gc。具体的判断方法如下所示：

{% highlight c %}

#define PyType_HasFeature(t,f)  (((t)->tp_flags & (f)) != 0)
#define Py_TPFLAGS_HAVE_GC (1UL << 14)

/* Test if a type has a GC head */
#define PyType_IS_GC(t) PyType_HasFeature((t), Py_TPFLAGS_HAVE_GC)

/* Test if an object has a GC head */
#define PyObject_IS_GC(o) (PyType_IS_GC(Py_TYPE(o)) && \
    (Py_TYPE(o)->tp_is_gc == NULL || Py_TYPE(o)->tp_is_gc(o)))


typedef int (*inquiry)(PyObject *);

typedef struct _typeobject {
    ...
    inquiry tp_is_gc; /* For PyObject_IS_GC */
    ...
}

{% endhighlight %}

CPython中为新创建的对象分配内存基本上都是由PyType_GenericAlloc函数完成的，通过跟踪这个函数可以看到CPython具体是如何为container object分配内存以及如何将contaner object放入链表中的，如下所示：

{% highlight c %}

#define _PyObject_GC_TRACK(o) do { \
    PyGC_Head *g = _Py_AS_GC(o); \
    if (_PyGCHead_REFS(g) != _PyGC_REFS_UNTRACKED) \
        Py_FatalError("GC object already tracked"); \
    _PyGCHead_SET_REFS(g, _PyGC_REFS_REACHABLE); \
    g->gc.gc_next = _PyGC_generation0; \
    g->gc.gc_prev = _PyGC_generation0->gc.gc_prev; \
    g->gc.gc_prev->gc.gc_next = g; \
    _PyGC_generation0->gc.gc_prev = g; \
    } while (0);

PyObject *
PyType_GenericAlloc(PyTypeObject *type, Py_ssize_t nitems)
{
    PyObject *obj;
    const size_t size = _PyObject_VAR_SIZE(type, nitems+1);
    /* note that we need to add one, for the sentinel */

    if (PyType_IS_GC(type))
        obj = _PyObject_GC_Malloc(size);
    else
        obj = (PyObject *)PyObject_MALLOC(size);
    ...
    if (PyType_IS_GC(type))
        _PyObject_GC_TRACK(obj);
    return obj;
}

PyObject *
_PyObject_GC_Malloc(size_t basicsize)
{
    return _PyObject_GC_Alloc(0, basicsize);
}

static PyObject *
_PyObject_GC_Alloc(int use_calloc, size_t basicsize)
{
    ...
    size = sizeof(PyGC_Head) + basicsize;
    if (use_calloc)
        g = (PyGC_Head *)PyObject_Calloc(1, size);
    else
        g = (PyGC_Head *)PyObject_Malloc(size);
    ....	
    generations[0].count++;
    ...
}

{% endhighlight %}

# 具体实现

---

回收算法的具体实现在Modules/gcmodule.c文件当中，具体代码就不贴了，仅仅将一些关键点列举如下：

1. 默认情况下，所有的container object分别归类到三个集合（generation）中，暂且称之为g0、g1与g2。g1 is older than g1 and g2 is older than g1。新创建的container object放在g0中。
2. 每个generation有threshold字段与count字段。对于g0而言，每创建一个container object，count加1。对于g1而言，每当g0进行了一次回收，count加1。g2的count增加规则类似于g1。
3. 在默认情况下，垃圾回收由创建container object时触发，触发条件是g0.count > g0.threshold。触发之后对count > threshold的oldest generation进行回收。
4. 对某个generation完成回收时，reachable object放到older generation中，对current generation和younger generation的count置0。
5. 对unreachable object的处理方法是除去所有对其他对象的引用，包括slots中的引用和__dict__中的引用，由subtype_clear方法完成。
6. 并不是所有的container object都会放在链表中，什么时候放入链表什么时候从链表剔除的逻辑更加复杂。
7. 为了安全考虑，定义了__del__方法的对象及其引用的对象不会被gc回收，这些对象需要人工处理。


# 小结

---

引用计数器和gc共同构成了CPython的对象回收机制，即使这样，依旧会有一些对象是无法自动回收的。

GIL一直是被认为是CPython无法真正利用CPU多核的原因，当了解到引用计数器后才明白，引用计数器是GIL存在的原因；现在了解到gc后才明白，对象回收机制是引用计数器存在的原因；而之所以没有更好的对象回收机制，则是C extension module的原因；C extension module的存在让释放GIL成为可能，同时提供了一个调用C／C++的机制，这让CPython在科学计算领域活得顺风顺水。所以，总的来说，GIL的问题并没有一个简单的答案。

(the end)
