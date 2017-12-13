---
layout: post
title:  "CPython：内存追踪"
date:   2017-12-13 21:00:00 +0800
categories: python
tags: python
---

CPython的内存管理已经比较完善，正常使用时基本上不存在内存泄漏的问题。但是Python作为一种动态语言，在简单的语法下隐藏了各种trick，如果使用不当或者各种滥用，也有可能给内存管理带来一定问题。所以，官方和社区维护有各种工具用于内存追踪（基本上都是针对CPython的），以备不时之需。这篇文章简要归纳一下每个工具的实现原理，通过这些原理可以推断出每个工具适用于什么场景。

# tracemalloc

---

tracemalloc是一个CPython自带的module，由[PEP 454](https://www.python.org/dev/peps/pep-0454/)提出，并在CPython 3.4引入。其底层由_tracemalloc提供支持，源码位于Modules/_tracemalloc.c。它的实现原理是将CPython默认的分配内存的函数换成了自己的函数，在自己的函数中实现相关的hook。

替换的关键部分为tracemalloc_alloc函数与tracemalloc_add_trace函数，如下所示：
{% highlight C %}
static void*
tracemalloc_alloc(int use_calloc, void *ctx, size_t nelem, size_t elsize)
{
    /* CPython默认的内存管理接口 */
    PyMemAllocatorEx *alloc = (PyMemAllocatorEx *)ctx;
    void *ptr;

    assert(elsize == 0 || nelem <= PY_SIZE_MAX / elsize);

    /* 正常分配内存 */
    if (use_calloc)
        ptr = alloc->calloc(alloc->ctx, nelem, elsize);
    else
        ptr = alloc->malloc(alloc->ctx, nelem * elsize);
    if (ptr == NULL)
        return NULL;

    /* 添加额外的信息 */
    TABLES_LOCK();
    if (tracemalloc_add_trace(ptr, nelem * elsize) < 0) {
        /* Failed to allocate a trace for the new memory block */
        TABLES_UNLOCK();
        alloc->free(alloc->ctx, ptr);
        return NULL;
    }
    TABLES_UNLOCK();
    return ptr;
}
{% endhighlight %}

对于申请的每一块内存，都会有一个trace记录该内存的信息。所有的trace都存在一张hash table中，也就是tracemalloc_traces，表中的key就是内存的指针。每个trace记录了内存的大小以及一串frame，frame中记录了从PyFrameObject提取的filename和lineno。至于tracemalloc提供的高级接口，都是围绕着tracemalloc_traces来处理的，例如Snapshot。tracemalloc_traces图示如下：

{% highlight text %}

tracemalloc_traces    one trace                          a list of frame
 +-------------+     +----------+                         +----------+
 |    ptr1     | --> |   size   |    one traceback   +--> | filename |
 +-------------+     +----------+     +--------+     |    |  lineno  |
 |    ptr2     |     | traceback| --> |  hash  |     |    +----------+
 +-------------+     +----------+     +--------+     |    | filename |
 |    ...      |                      | nframe |     |    |  lineno  |
 +-------------+                      +--------+     |    +----------+
 |    ...      |                      | frames | ----+    |    ...   |
                                      +--------+

{% endhighlight %}

由于直接介入了CPython的内存管理接口，所以只要是使用了CPython内存接口的行为都可以跟踪到。由于采用了同步介入（而不是异步统计）的方式，所以可以通过读取PyFrameObject跟踪到源码的位置。基于这两个特点，进行diff操作可以说是非常准确的。

对于没有使用CPython内存接口的行为是无能为力的，这种行为也是官方不推荐的。

# pympler

---

todo

(the end)
