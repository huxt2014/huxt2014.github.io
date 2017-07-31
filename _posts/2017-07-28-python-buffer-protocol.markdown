---
layout: post
title:  "深入Python：Buffer Protocol和Numpy实现"
date:   2017-07-28 21:00:00 +0800
categories: python
---

最近在使用SQLAlchemy+mysqlclient+Pandas进行数据处理的时候发现了一个比较尴尬的问题，那就是Python太吃内存了。按理说不应该这样的，因为Pandas底层用的是Numpy，已经是直接操作内存块而不是操作Python对象了，并且mysqlclient的关键模块是用C写的。但实际情况是从数据库读取600W条记录（4个字段，VARCHAR * 3+BIGINT * 1）时，内存使用量超过了2G。

稍微看了一下各个接口的源码后，发现问题在于数据转换的过程中。虽然mysqlclient与Numpy的关键部分都是用C实现的，但是数据从mysqlclient传到Numpy的过程中用的是Python对象。这个过程除了占用内存，还会增加额外的计算量（C->Python->C）。

有没有可能对这一过程进行优化，跳过将数据转化成Python对象这一步呢。直接的方案就是从mysqlclient直接拿到numpy.ndarray了，但是这对numpy有依赖。再仔细考虑一下，发现一个可能可行的方案：Python的Buffer Protocol。

# 现象回顾

---

为了观察到内存的使用情况，这里除了使用Mac系统自带的Active Monitor查看外，还使用到了Python第三方module：pympler。为了简化分析，就只读取BIGINT那一列了。

{% highlight ipython %}

In [7]: from pympler import tracker

In [8]: tr = tracker.SummaryTracker()

In [12]: rs = engine.execute('select traffic from my_table limit 3000000')

In [13]: tr.print_diff()
                      types |   # objects |   total size
=========================== | =========== | ============
  <class 'collections.deque |          41 |     25.82 KB
               <class 'dict |          42 |     25.25 KB
                <class 'str |         247 |     24.98 KB
               <class 'type |           2 |      5.98 KB

In [14]: records = rs.fetchall()

In [15]: tr.print_diff()
                                         types |   # objects |   total size
============================================== | =========== | ============
     <class 'sqlalchemy.engine.result.RowProxy |     3000000 |    137.33 MB
                                  <class 'list |          44 |     23.95 MB
  <class 'prompt_toolkit.utils._CharSizesCache |           0 |     12.00 KB
                                  <class 'dict |           3 |      2.44 KB
                                   <class 'str |          34 |      1.79 KB

{% endhighlight %}

当执行SQL后得到rs的时候，Active Monitor中显示已经使用了300M左右的内存了，但是pympler并没有检测出来。这应该是因为数据库的driver是用C写的，driver用了自己的接口动态获取内存而没有调用Python的接口。到这一步，从数据库中拿到的数据还在driver所控制的内存中。

执行rs.fetchall()后，mysqlclient将数据转化为Python对象吐了出来，通过pympler可以看到占用了超过100M的内存。

MySQL的BIGINT占用的字节数是8 bytes，所以从理想状况来看，存放300W个BIGING根本不需要超过100M的内存，而是仅仅需要不到3M的内存，从Numpy的情况来看貌似也是这样。不得不吐槽一下，MySQL的driver是怎么用到300M内存的……

{% highlight ipython %}

In [27]: arr = np.ndarray(shape=(3000000,), dtype=np.int8)

In [28]: sys.getsizeof(arr)
Out[28]: 3000096

In [29]: arr.nbytes
Out[29]: 3000000

In [30]: tr.print_diff()
                  types |   # objects |   total size
======================= | =========== | ============
  <class 'numpy.ndarray |           1 |      2.86 MB
          <class 'tuple |          21 |      1.34 KB
            <class 'str |          13 |      1.03 KB

{% endhighlight %}

# Buffer Protocol

---

为了暴露对象底层的内存，Python提供了Buffer Protocol，有了这样一个统一的接口，就可以方便地在C的层面转化和操作对象了。Buffer Protocal的介绍和接口说明在官方文档里面都有，在这里就不赘述了，下面先来看看Numpy如何利用到这个接口。

{% highlight ipython %}

In [35]: block = bytearray(b'abcdef')

In [36]: arr = np.ndarray(shape=(6, ), dtype=np.int8, buffer=block)

In [37]: arr
Out[37]: array([ 97,  98,  99, 100, 101, 102], dtype=int8)

In [38]: arr[0] = 98

In [39]: arr.shape = (3, 2)

In [40]: arr[2,1] = 122

In [41]: block
Out[41]: bytearray(b'bbcdez')

In [42]: block[0] = 122

In [43]: arr
Out[43]:
array([[122,  98],
       [ 99, 100],
       [101, 122]], dtype=int8)

{% endhighlight %}

从上面的示例可以看到，实际上arr与block指向的是同一段内存，对其中一个进行更改都会影响到另外一个。至于Numpy是如何实现的，那么就可以看看Numpy的源码了。从github clone到源码后，可以看到如下的关键部分。

{% highlight c %}
/* numpy/numpy/core/src/multiarray/arrayobject.c */

static PyObject *
array_new(PyTypeObject *subtype, PyObject *args, PyObject *kwds)
{
    static char *kwlist[] = {"shape", "dtype", "buffer", "offset", "strides",
                             "order", NULL};
    PyArray_Descr *descr = NULL;
    int itemsize;
    PyArray_Dims dims = {NULL, 0};
    PyArray_Dims strides = {NULL, 0};
    PyArray_Chunk buffer;
    npy_longlong offset = 0;
    NPY_ORDER order = NPY_CORDER;
    int is_f_order = 0;
    PyArrayObject *ret;

    buffer.ptr = NULL;

    if (!PyArg_ParseTupleAndKeywords(args, kwds, "O&|O&O&LO&O&:ndarray",
                                     kwlist, PyArray_IntpConverter,
                                     &dims,
                                     PyArray_DescrConverter,
                                     &descr,
                                     PyArray_BufferConverter,
                                     &buffer,
                                     &offset,
                                     &PyArray_IntpConverter,
                                     &strides,
                                     &PyArray_OrderConverter,
                                     &order)) {
        goto fail;
    }

...

    if (buffer.ptr == NULL) {
        ...
    }
    else {
        ...
        ret = (PyArrayObject *)\
            PyArray_NewFromDescr_int(subtype, descr,
                                     dims.len, dims.ptr,
                                     strides.ptr,
                                     offset + (char *)buffer.ptr,
                                     buffer.flags, NULL, 0, 1);
        ...
        /* 需要获得引用 */
        Py_INCREF(buffer.base);
    ...
    }
    ...
    return (PyObject *)ret;
}
{% endhighlight %}

{% highlight c %}
/* numpy/numpy/core/src/multiarray/conversion_utils.c */

NPY_NO_EXPORT int
PyArray_BufferConverter(PyObject *obj, PyArray_Chunk *buf)
{
    Py_buffer view;
    ...

    if (PyObject_GetBuffer(obj, &view, PyBUF_ANY_CONTIGUOUS|PyBUF_WRITABLE) != 0) {
        PyErr_Clear();
        buf->flags &= ~NPY_ARRAY_WRITEABLE;
        if (PyObject_GetBuffer(obj, &view, PyBUF_ANY_CONTIGUOUS) != 0) {
            return NPY_FAIL;
        }
    }

    buf->ptr = view.buf;
    buf->len = (npy_intp) view.len;

    /* 需要释放引用 */
    PyBuffer_Release(&view);

    /* Point to the base of the buffer object if present */
    if (PyMemoryView_Check(obj)) {
        buf->base = PyMemoryView_GET_BASE(obj);
    }
    ...

    if (buf->base == NULL) {
        buf->base = obj;
    }
    return NPY_SUCCEED;
}

{% endhighlight %}

PyArray_Chunk是Numpy自己定义的type，如果np.ndarray中传递了buffer，那么会通过PyArray_BufferConverter函数将buffer的相关信息存到PyArray_Chunk中。PyArray_BufferConverter函数调用了Python的PyObject_GetBuffer。

在使用Python C API的时候，需要特别注意引用的处理。arry_new调用了Py_INCREF(buffer.base)来获得了原始对象的引用，如果原始对象被回收而array对象还存在就会出问题了。同时在PyArray_BufferConverter中也调用了PyBuffer_Release(&view)，这个是必要的，因为在调用PyObject_GetBuffer的时候会获得原始对象的引用。

Numpy除了可以使用其他对象的内存，还通过Buffer Protocol向外部暴露了自己的内存，通过Python的内置对象
memoryview可以看看效果。

{% highlight ipython %}

In [68]: arr.shape=(6,)

In [69]: v = memoryview(arr)

In [70]: v.shape, v.nbytes, v.readonly
Out[70]: ((6,), 6, False)

In [71]: v[0] = 0

In [72]: arr
Out[72]: array([  0,  98,  99, 100, 101, 122], dtype=int8)

{% endhighlight %}

支持Buffer Procotol来暴露内存需要定义PyBufferProcs结构体，并塞到tp_as_buffer中，这个结构体里面存放需要的函数指针。官方文档阐述了对应函数中必不可少的步骤，而Numpy给出了实现样例。

{% highlight c %}
# numpy/numpy/core/src/multiarray/arrayobject.c

NPY_NO_EXPORT PyTypeObject PyArray_Type = {
    ...
    &array_as_buffer,                           /* tp_as_buffer */
    ...
};

{% endhighlight %}

{% highlight c %}
# numpy/numpy/core/src/multiarray/buffer.c

static int
array_getbuffer(PyObject *obj, Py_buffer *view, int flags)
{
    PyArrayObject *self;
    _buffer_info_t *info = NULL;

    self = (PyArrayObject*)obj;

    ...

    view->buf = PyArray_DATA(self);
    view->suboffsets = NULL;
    view->itemsize = PyArray_ITEMSIZE(self);
    view->readonly = !PyArray_ISWRITEABLE(self);
    view->internal = NULL;
    view->len = PyArray_NBYTES(self);

    ...
    view->obj = (PyObject*)self;

    /* 需要返回一个新的引用 */
    Py_INCREF(self);
}

NPY_NO_EXPORT PyBufferProcs array_as_buffer = {
#if !defined(NPY_PY3K)
    (readbufferproc)array_getreadbuf,       /*bf_getreadbuffer*/
    (writebufferproc)array_getwritebuf,     /*bf_getwritebuffer*/
    (segcountproc)array_getsegcount,        /*bf_getsegcount*/
    (charbufferproc)array_getcharbuf,       /*bf_getcharbuffer*/
#endif
    (getbufferproc)array_getbuffer,
    (releasebufferproc)0,
};
{% endhighlight %}

# 小结

---

回到Python读取数据库的问题，在读取大量的数据的时候，mysqlclient的性能还是比较捉急的（即使底层用C实现）。如果能够支持Buffer Protocol，至少就可以不需要转换到Python对象再喂给Numpy了。虽然Numpy实现的Buffer Protocol可以作为一个不错的参考，但是还有很多东西需要考虑，例如与MySQL driver的交互，以及数据类型的问题。
