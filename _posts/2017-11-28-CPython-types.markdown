---
layout: post
title:  "CPython：类型系统"
date:   2017-11-28 21:00:00 +0800
categories: python
tags: python
---

Python作为一种动态语言，使用起来可谓是相当灵活。我们可以像执行脚本一样来执行一个Python文件，也可以用Python来实现OOP风格的中小型项目，此外Python在一定程度上支持函数式编程。这么多灵活的用法，一方面得益于动态语言的特点，另外一方面得益于Python强大的类型系统。

在Python中，几乎所有的东西都是对象。Python使用metaclass的方法为这些对象提供了行为规范，在这基础上实现了OOP和函数式编程的支持。这片文章主要从OOP的角度来看一下CPython中类型系统是如何实现的。

# metaclass与object

---

对Python已经有基本了解的人应该知道object。所有内置类型（例如int）以及用户自定义的class都会默认继承object，例如int.\_\_bases\_\_ == (object,)。这一点从OOP的角度来讲很容易理解。但是，如果说class和instance都是object的实例，例如isinstance(int, object) == True，isinstance(1, object) == True，那么就感觉有点奇怪了。

此外，还有一个容易令人疑惑的东西，那就是type。之所以称之为“东西”，是因为type在不同的场合下有不同的含义，乍眼一看还真不知道这是怎样一种存在。有时我们将其视为一种描述，例如dict是一种type；同时，用type可以判断instance属于哪个class，例如type(1) == int；此外，type又像一个class，可以被继承，也可以产生instance；更令人诧异的是，type产生的一个instance是一个class。

如果说object与type已经够令人迷惑了，那再追寻一下object与type的关系，就更令人抓狂了：type(object) == type，isinstance(type, object) == True，type.\_\_bases\_\_ == (object,)。这些都是什么鬼！

是的，我们可以通过阅读CPython源码来找到上述现象产生的原因，例如看看PyObject、PyTypeObject类型是如何实现的，观察一下ob_type、ob_bases字段的作用。但是，对于我来说，了解这些后依旧无法完全消除我心中的疑惑。就像面对一台复杂的机器，我能通过研究它的结构来了解它是怎么工作的，但是我不知道它是被如何设计成这样的。这种感觉一直无法消除，直到我阅读了UML规范。

简单来看，UML就是一堆各式各样的方框、箭头，用这些元素可以描述一个系统。但是仅有一些构图元素是不够的，只有为每个元素赋予特定的意义，才能用来表述一个系统。这些描述元素意义的信息，就称为metamodel。在UML2.0版本中，描述metamodel的方法不太直观，理解起来也有点麻烦。在UML2.5中，metamodel由UML进行表述！乍看之下比较意外，因为metamodel是用来规范UML的，但是它居然由UML表述。但实际情况是，使用UML表述后，metamodel变得更容易理解了。这种思路不是仅此一家，例如PyPy可以用来运行Python代码，而它自己是用Python实现的，以及CPython的类型系统。

在CPython中，所有东西都以对象的形式存在，class是对象，instance是对象，function也是对象。这些东西之所以被分类为class、instance、function，是因为它们体现的行为不同。既然存在不同的行为，那么就要有东西来定义、描述这些行为，这个东西就是metaclass。那metaclass以什么样的形式存在呢，没错，以对象的形式存在。这个逻辑便是CPython构建对象系统的基本思路，如下所示：

{% highlight text %}
                                   object
               +---------------------+----------------------+
               |                     |                      |
           metaclass                class                instance
     +------------------+--------------------------+----------------+
     |      type        |    object int dict       |    1, {}, ...  |
     |                  |    function method  ...  |   def  lambda  |
     | subclass_of_type |    user_defined_class    |                |
     +------------------+--------------------------+----------------+
             |               |              |                |
             +--as meta-->>--+              +---as meta-->>--+

{% endhighlight %}

值得一提的是，虽然object、int等内置类型是class，但是它们描述了instance的行为，所以在C层面，type、object、int都是PyTypeObject。

有了这个基础后，定义各种对象的行为就顺其自然了，例如继承关系，运算操作、产生instance、函数的行为等等。此外，理解了这个思路后，就没必要再纠结某些特殊情况下调用type()、isinstance()所得到的结果是什么了，因为对于这些特殊情况，它们的结果只是正好被实现为这样而已。

# 类型系统的初始化

----

在在解释器初始化的过程中会完成类型系统初始化的工作，对应的代码位置在Py_Initialize函数中的_Py_ReadyTypes调用，而_Py_ReadyTypes里面全部是PyType_Ready调用，如下所示。

{% highlight c %}

/* Objects/object.c */

void
_Py_ReadyTypes(void)
{
    if (PyType_Ready(&PyBaseObject_Type) < 0)
        Py_FatalError("Can't initialize object type");

    if (PyType_Ready(&PyType_Type) < 0)
        Py_FatalError("Can't initialize type type");
    ...
}

{% endhighlight %}

可以看到，最先初始化的是PyBaseObject_Type（也就是object），然后是PyType_Type（也就是type）。此外，内置的类型的内存都是静态分配的。

PyType_Ready函数对PyTypeObject类型的变量填充了各种东西，例如ob_type、tp_base、tp_bases、tp_dict等等。由于内容众多，逻辑复杂，要全部理顺还是比较麻烦的。为了对初始化的过程有一个快速的认识，我们可以直接观察初始化后的结果。随便写了一个C的module，观测结果如下：
{% highlight ipython %}

In [1]: import basic_func

In [2]: basic_func.inspect_type(object)
ob_type: <class 'type'>
tp_base: NULL
tp_bases: ()
tp_call: NULL
tp_alloc: PyType_GenericAlloc
tp_new: object_new
tp_init: object_init
tp_getattro: PyObject_GenericGetAttr

In [3]: basic_func.inspect_type(type)
ob_type: <class 'type'>
tp_base: <class 'object'>
tp_bases: (<class 'object'>,)
tp_call: type_call
tp_alloc: PyType_GenericAlloc
tp_new: type_new
tp_init: type_init
tp_getattro: type_getattro

In [4]: basic_func.inspect_type(list)
ob_type: <class 'type'>
tp_base: <class 'object'>
tp_bases: (<class 'object'>,)
tp_call: NULL
tp_alloc: PyType_GenericAlloc
tp_new: unknown
tp_init: unknown
tp_getattro: PyObject_GenericGetAttr

{% endhighlight %}

# 生成class

---

在Python中生成class的方法有三种：在Python代码中定义class，调用metaclass，在c extension module中定义。第三种方法和Python内置类型的定义过程是一样的，相对直观，就不赘述了，这里主要来看看更有趣的前面两种。

在Python代码中定义class的过程可以从字节码开始追溯，如下所示：

{% highlight python %}

# Python 代码
class MyClass:
    pass

# 编译后的字节码
 0 LOAD_BUILD_CLASS
 1 LOAD_CONST               0 (<code object MyClass at 0x10438ef60 ...>)
 4 LOAD_CONST               1 ('MyClass')
 7 MAKE_FUNCTION            0
10 LOAD_CONST               1 ('MyClass')
13 CALL_FUNCTION            2 (2 positional, 0 keyword pair)
16 STORE_NAME               0 (MyClass)
19 LOAD_CONST               2 (None)
22 RETURN_VALUE

{% endhighlight %}

{% highlight C %}
/* LOAD_BUILD_CLASS 对应的C代码*/
        TARGET(LOAD_BUILD_CLASS) {
            _Py_IDENTIFIER(__build_class__);

            PyObject *bc;
            if (PyDict_CheckExact(f->f_builtins)) {
                bc = _PyDict_GetItemId(f->f_builtins, &PyId___build_class__);
               ...
            }
            ...
            PUSH(bc);
        ...
        }

/* CALL_FUNCTION对应的C代码 */
        TARGET(CALL_FUNCTION) {
            PyObject **sp, *res;
            PCALL(PCALL_ALL);
            sp = stack_pointer;
            res = call_function(&sp, oparg);
            stack_pointer = sp;
            PUSH(res);
            ...
        }

/* __build_class__ 的C实现 */
static PyObject *
builtin___build_class__(PyObject *self, PyObject *args, PyObject *kwds)
{
    /* 先从kwds中找meta */
    ...
    /* 如果没有找到*/
    if (meta == NULL) {
        /* if there are no bases, use type: */
        if (PyTuple_GET_SIZE(bases) == 0) {
            meta = (PyObject *) (&PyType_Type);
        }
        /* else get the type of the first base */
        else {
            PyObject *base0 = PyTuple_GET_ITEM(bases, 0);
            meta = (PyObject *) (base0->ob_type);
        }
        Py_INCREF(meta);
        isclass = 1;  /* meta is really a class */
    }
    ...
    /* 最后会执行这个*/
    cls = PyEval_CallObjectWithKeywords(meta, margs, mkw);
    ...
    return cls
}

PyObject *
PyObject_Call(PyObject *func, PyObject *arg, PyObject *kw)
{
    ...
    call = func->ob_type->tp_call;
    ...
}

{% endhighlight %}

通过调用metaclass来生成class的过程也可以从字节码开始追溯，如下所示：

{% highlight python %}
# Python 代码
type("MyClass", (), {})

# 编译得到的字节码
 0 LOAD_NAME                0 (type)
 3 LOAD_CONST               0 ('MyClass')
 6 BUILD_TUPLE              0
 9 BUILD_MAP                0
12 CALL_FUNCTION            3 (3 positional, 0 keyword pair)
15 POP_TOP
16 LOAD_CONST               1 (None)
19 RETURN_VALUE

# 接下来不用再看了，和上面的追溯过程是一样的

{% endhighlight %}

可以看到，不管是通过定义来得到class，还是通过调用metaclass来得到class，最终生成class的方法都是一样的，那就是tp_call函数指针。在type中，tp_call指向的是Objects/typeobject.c文件中定义的type_call函数。理解这个函数后，就可以理解调用type时产生的各种结果了：

1. 判断对象的类型。例如，type(1) == int。
2. 生成class。例如，type("MyClass", (object,), {})。
3. 帮class生成instance。例如type.\_\_call\_\_(int, 1)。

# 生成instance

---

上文说到，type_call函数即能生成class，也能帮class生成instance，这里面的原因在于type_call会调用tp_new，metaclass和class的tp_new是不一样的。此外，对于生成instance，type_call还会调用tp_init，也就是object_init，如下所示：

{% highlight ipython %}

In [2]: basic_func.inspect_type(type)
ob_type: <class 'type'>
tp_base: <class 'object'>
tp_bases: (<class 'object'>,)
tp_call: type_call
tp_alloc: PyType_GenericAlloc
tp_new: type_new
tp_init: type_init
tp_getattro: type_getattro

In [3]: basic_func.inspect_type(MyClass1)
ob_type: <class 'type'>
tp_base: <class 'object'>
tp_bases: (<class 'object'>,)
tp_call: NULL
tp_alloc: PyType_GenericAlloc
tp_new: object_new
tp_init: object_init
tp_getattro: PyObject_GenericGetAttr

{% endhighlight %}

上面显示的object_new、object_init函数从object继承，如果在定义class的时候有定义__init__和__new__，自定义的会填充到tp_new与tp_init中。可以说，对于生成instance过程中的一切问题，都可以在type_call中找到答案。

# 内存中的对象

---

既然已经知道了CPython中三种主要的对象（metaclass、class和instance）和它们之间的关系，那么接下来就可以近一步来看看它们在内存中是怎样的。考虑到PyTypeObject的字段茫茫多，不可能挨个看过去，这里主要来看看和__dict__、__slots__相关的内容，因为这与CPython使用内存的效率关系密切。

用户在定义class的时候可以为其添加自定义的attribute，在生成instance的时候，instance也会继承这些attribute。默认情况下，这些attribute都是放在__dict__中的，所谓的__dict__其实就是一个dict。进一步说，class中有一个指针指向一个dict，每个instance中有一个指针指向各自的dict。需要注意的是，CPython是用哈希表来实现dict的，这里面会有一部分内存是不存东西的。为了提高内存的使用率，__slots__被引入了进来。

要理解__slots__机制，就需要对class在内存中的结构有更深一步的了解。上文提到，每个class在内存中都对应了一个PyTypeObject变量，其实这个描述并不准确，实际上还要加一个尾巴，如下所示：

{% highlight C %}

/* The *real* layout of a type object when allocated on the heap */
typedef struct _heaptypeobject {
    PyTypeObject ht_type;
    PyAsyncMethods as_async;
    PyNumberMethods as_number;
    PyMappingMethods as_mapping;
    PySequenceMethods as_sequence;
    PyBufferProcs as_buffer;
    PyObject *ht_name, *ht_slots, *ht_qualname;
    struct _dictkeysobject *ht_cached_keys;
    /* here are optional user slots, followed by the members. */
} PyHeapTypeObject;

{% endhighlight %}

当CPython创建一个class为其分配内存的时候，最少会分配sizeof(PyHeapTypeObject) + nslots*sizeof(PyMemberDef)大小的内存（之所以有“最少”，是因为还有一些辅助性的空间）。如果有__slots__的话，会以PyMemberDef的形式记在尾部。相对应的，这个class生成的instance中将不会再有__dict__。

一图胜千言，两种内存分配方式如下所示：

{% highlight text %}

class Class1:
    a = 1
    b = 2
    c = 3

+-------------------+
|ob_refcnt          |
|...                |
|tp_basicsize       | = base->basicsize + 2*sizeof(PyObject *)
|...                |
|tp_weaklistoffset  | = base->basicsize + sizeof(PyObject *)
|...                |
|tp_dictoffset      | = base->basicsize
|...                |
|ht_cached_keys     |
+-------------------+

ins = Class1()

+-----------+
|gc_next    |
|gc_prev    |   PyGC_Head
|gc_refs    |
+-----------+
|ob_refcnt  |
|ob_type    | -> Class1
|PyObject * | -> dict
|PyObject * | -> weaklist
+-----------+


{% endhighlight %}


{% highlight text %}

class Class2:
    __slots__ = ('a','b','c')

+-------------------+
|ob_refcnt          |
|...                |
|tp_basicsize       | = base->basicsize + nslot*sizeof(PyObject *)
|...                |
|tp_weaklistoffset  | = 0
|...                |
|tp_dictoffset      | = 0
|...                |
|ht_cached_keys     |
+-------------------+
|PyMemberDef for a  |
|PyMemberDef for b  |
|PyMemberDef for c  |
+-------------------+

ins = Class2()

+-----------+
|gc_next    |
|gc_prev    |   PyGC_Head
|gc_refs    |
+-----------+
|ob_refcnt  |
|ob_type    | -> Class2
|PyObject * | -> a
|PyObject * | -> b
|PyObject * | -> c
+-----------+


{% endhighlight %}


(the end)
