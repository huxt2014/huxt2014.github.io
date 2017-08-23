---
layout: post
title:  "深入Python：内置dict类型的实现"
date:   2017-08-22 21:00:00 +0800
categories: python
tags: python
---

Python对内置dict类型的使用程度非常深，kwargs arguments，class method lookup，instance attribute lookup等等都是基于内置dict实现的，可以说dict的性能对Python性能有着举足轻重的影响。Python的内置dict类型基于hash表实现，这篇文章在简单回顾hash表的设计要点的基础上，深入探究一下Python内置dict类型是如何实现的。

# hash表回顾与Python选择

---

hash表是一种可以根据key来操作item的数据结构，其最基本的三种操作为插入、查找、删除。hash表的存储结构通常为数组，当根据key来操作item时，通过一个hash函数将key转化为数组中的位置，这样根据key定位到item的平均时间复杂度为O(1)。hash表中需要解决的最基本问题有两个：如何选择hash函数使得item能够比较均匀地分布在数组当中，以及如何解决不同的key对应到了数组中的同一个slot。

#### 1. hash函数

一个好的hash函数能够将key比较均匀地映射到数组的所有slot中，为了达到这样的效果理论上需要结合key的统计分布来设计。实践当中，也可以忽略key的统计分布来设计比较通用的hash函数，这样的设计思路通常有两种：一种是基于除法的， ![](/images/python-dict/1.png) ，其中m为数组长度，其选择一般避免2的指数幂，倾向于质数；另外一种是基于乘法的， ![](/images/python-dict/2.png) ，其中0<A<1。

hash函数的具体的实现与解决冲突的方法高度相关，对于Python而言，在考虑hash函数的具体实现之前，先要决绝key的问题。因为在同一个dict实例中，不同类型的对象都可以当作是key，所以这里面需要有一个转化的过程。实际上，Python的数据类型可以有一个计算该类型实例hash值的函数，记录在各自的PyTypeObject中，这个hash值就可以替代实例作为dict中的key。如果该函数没有被设置，那么说明该类型不支持计算hash，因此其实例也无法作为dict中的key。

#### 2. 解决冲突

解决冲突的方法通常有两个，chaining和open addressing。

chaining方法中，数组中存储的是链表的头节点，item存储在链表中。当不同的key落到了同一个slot中时，其对应的item就可以存在同一个链表当中了。在这种实现种，插入的时间复杂度是O(1)，查找的时间复杂度是O(1+a)，a为平均存储率。

open addressing方法中，通过hash函数和一定的规则可以根据key产生一组值（probe sequence），每个值对应了slot的位置，根据这一组值按顺序去查找slot。在插入操作中，将item插入到第一个未被占用的slot；在查找操作中，如果slot被占用了则比较item，直到找到相同的item或者发现未被占用的slot；在删除操作中，需要将清空的slot作额外的标记，例如插入一个dummy item。产生probe sequence的常用方法有三种：

1. linear probing: ![](/images/python-dict/3.png)
2. quadratic probing: ![](/images/python-dict/4.png)
3. double hashing: ![](/images/python-dict/5.png) 

Python选择了open addressing的方式，原因是chaining需要频繁地动态分配/回收内存。其产生probe sequence的方式采用了double hashing。

# Python实现

---

#### 1. 数据结构

dict类型基础的数据结构如下所示。PyDictObject中，ma_used记录了dict实例中实际存储item的个数，用len查看dict实例长度的时候返回的就是这个值；ma_keys指向了一个PyDictKeysObject，这个对象就是存储key和item的地方；ma_values在后面细说。PyDictKeysObject中，dk_size记录了数组的容量；dk_usable中记录了还有多少slot能够使用；dk_entries就是数组的地址了。

{% highlight C %}

# Include/pyport.h
typedef Py_ssize_t Py_hash_t;


# Objects/dict-common.h
typedef struct {
    Py_hash_t me_hash;  /* hash值 */
    PyObject *me_key;   /* key */
    PyObject *me_value; /* value，仅在combined tables中有效 */
} PyDictKeyEntry;


typedef PyDictKeyEntry *(*dict_lookup_func)
(PyDictObject *mp, PyObject *key, Py_hash_t hash, PyObject ***value_addr);


struct _dictkeysobject {
    Py_ssize_t dk_refcnt;          /* 帮助标示是combined还是split模式 */
    Py_ssize_t dk_size;            /* 总长度，新创建的dict初始值为
                                      PyDict_MINSIZE_COMBINED */
    dict_lookup_func dk_lookup;
    Py_ssize_t dk_usable;          /* 剩余空间，初始化为USABLE_FRACTION(dk_size)，
                                      即(2*dk_size+1)/3 */
    PyDictKeyEntry dk_entries[1];  /* 数组的地址 */
};


# Include/dictobject.h
typedef struct _dictkeysobject PyDictKeysObject;

typedef struct {
    PyObject_HEAD
    Py_ssize_t ma_used;
    PyDictKeysObject *ma_keys;
    PyObject **ma_values;
} PyDictObject;


{% endhighlight %}

PyDictObject中有一个ma_values值，是一个数组的地址，数组中存了PyObject *。之所以会有这个，是因为在CPython 3.3版本之后，Python的dict类型在内部有两种存储模式：combined（旧）和split（新），ma_values就是用来支持split模式的。

Python 3.3之后，在使用dict或者{}得到dict实例的时候，得到的都是combined模式的dict，key和item都存在ma_keys中。在所有实例创建__dict__的时候，使用的是split模式。split模式中，所有实例的ma_keys都是共享的，共享的ma_keys存储在对应的PyTypeObject中，这样就不需要为每个实例都分配ma_keys的内存了。在OOP风格的程序中，这种实现可以节省10%到20%的内存，以及少量的性能提升。

split模式是为__dict__以及OOP的继承／多态设计的，其内部实现也有一些额外的限制。例如，key只能是unicode，某些操作会使得split模式的dict对象转变为combined模式。了解了split这种实现方式之后，如下Python代码是怎么执行的也能猜个大概了。如果能够避免split模式转变为combined，也就相当于在内存方面进行了优化。

{% highlight C %}

import sys

class Test:
    a = 1

    def __init__(self, a):
        # stored in ma_values, no extra ma_keys
        self.a = a


t1 = Test(2)
print("reues keys: size = ", sys.getsizeof(t1.__dict__))

t1.a = 10
print("reuse keys: size = ", sys.getsizeof(t1.__dict__))

t1.__dict__[1] = 1
print("translate to combined: size = ", sys.getsizeof(t1.__dict__))

# output:
# reues keys: size =  96
# reuse keys: size =  96
# translate to combined: size =  288
{% endhighlight %}

#### 2. 初始容量与调整容量

hash表的基础操作时间复杂度在O(1)，这是忽略了容量调整的情况下。容量调整需要重新分配／回收内存，以及将item重新插入到hash表中，如果过于频繁地调整容量，对性能也是一种损耗。Python的很多内置操作都是基于hash表的，包括但不仅限于：

1. Passing keyword arguments, typically 1 to 3 elements, read and one write
2. Class method lookup, typically 8 to 16 elements，usually written once with many lookups
3. Instance attribute lookup and Global variables, typically 4 to 10 elements，both reads and writes are common
4. Builtins, about 150 interned strings, frequent reads and almost never written.

此外，dict也是在应用程序中经常使用到的数据类型，可能存储任意容量的数据以及各种操作便好。所以，Python实现内置dict类型的时候会更多地考虑到通用性，然后兼顾一些特殊情况。在初始容量和容量调整方面也是如此。

对于combined模式和spit模式的dict，默认的初始大小（dk_size）分别为8和4。对于可使用容量（dk_usable）的设置为(2*dk_size+1)/3，即数组大小的2/3。当dk_usable为0时，再新增item就会触发resize了，调整后的大小如下所示：

{% highlight C %}

/* dk_used*2 + dk_size/2 */
#define GROWTH_RATE(d) (((d)->ma_used*2)+((d)->ma_keys->dk_size>>1))

static int
insertion_resize(PyDictObject *mp)
{
    return dictresize(mp, GROWTH_RATE(mp));
}

static int
dictresize(PyDictObject *mp, Py_ssize_t minused)
{
    ...

    for (newsize = PyDict_MINSIZE_COMBINED;
         newsize <= minused && newsize > 0;
         newsize <<= 1)
        ;

    ....

    /* 可以看到，数组大小都是2的指数 */
    mp->ma_keys = new_keys_object(newsize);

    ...

    for (i = 0; i < oldsize; i++) {
        PyDictKeyEntry *ep = &oldkeys->dk_entries[i];
        if (ep->me_value != NULL) {
            assert(ep->me_key != dummy);
            insertdict_clean(mp, ep->me_key, ep->me_hash, ep->me_value);
        }
    }

    ...

}

{% endhighlight %}

不仅仅是insert时容量不够会触发resize，很多其他情况下也会触发resize，这里就不一一列举了。如果操作dict的时候遇到性能问题，可以从C源码去核实一下，有没有触发不必要的resize。

值得一提的是，dummy item会占用剩余空间。即当pop一个元素时，dk_usable不会减1，当insert一个元素且该元素落到dummy item上时，dk_usable不会加1。

#### 3. 解决冲突

根据具体情况不同，dict为查找操作定义了四种实现：lookdict，lookdict_unicode，lookdict_unicode_nodummy，lookdict_split。这四种实现中计算hash和解决冲突的方式都是一样的，所以拿通用的lookdict函数来看一下其中的关键部分。

{% highlight C %}
#define PERTURB_SHIFT 5
#define DK_MASK(dk) (((dk)->dk_size)-1)

static PyDictKeyEntry *
lookdict(PyDictObject *mp, PyObject *key,
         Py_hash_t hash, PyObject ***value_addr)
{
    ...

    mask = DK_MASK(mp->ma_keys);
    ep0 = &mp->ma_keys->dk_entries[0];
    i = (size_t)hash & mask;
    ep = &ep0[i];

    ...

    # search probe sequence
    for (perturb = hash; ; perturb >>= PERTURB_SHIFT) {
        /* (5*i) + 1 + perturb */
        i = (i << 2) + i + perturb + 1;
        ep = &ep0[i & mask];
        ...
    }
}
{% endhighlight %}

比较有意思的是，虽然选择hash函数时的通常做法是避免取2\*\*i（因为这样相当于只考虑了最低的i位），但是Python的实现中dict的长度一定是2\*\*i，且hash函数的实现就是取的最低的i位。选择这样的实现方式是基于以下三点综合考虑的：

1. 计算足够快
2. 特殊情况没那么重要
3. 有足够好的计算probe sequence的方法

计算probe sequence的方法从代码中可以看出来，相当于是double hashing。稍微验证一下之后发现，这种方法产生的probe sequence确实可以遍历整个数组，八成是有经过数学证明的。

#### 参数调整

内置dict类型更多考虑的是通用情况，在经过大量性能测试之后，选择了一个各个方面都比较折中的实现方法，在实际的使用当中其实还是有调整的空间的。源码的dictnotes.txt文件中有相关的介绍，例如PyDict_MINSIZE_SPLIT、 PyDict_MINSIZE_COMBINED、USABLE_FRACTION和GROWTH_RATE四个参数都是可以稍微调整的。但是其中也提到，调整之后对某些场景下的性能有提升，但是会降低另外一些场景下的性能，所以调整的时候一定要经过足够全面的性能测试。

(the end)
