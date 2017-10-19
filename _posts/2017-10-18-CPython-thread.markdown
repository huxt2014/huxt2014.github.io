---
layout: post
title:  "CPython：线程模型与GIL"
date:   2017-10-18 21:00:00 +0800
categories: python
tags: python
---

Python支持线程级别的并发，在CPython中，每个Python线程都对应了一个操作系统的原生线程。虽然如此，但是因为GIL的存在，默认情况下CPython也无法充分利用CPU多核来完成并行。然而，在满足一定条件下，GIL是可以被释放的，释放GIL后Python的线程能够真正意义上的利用多核实现并行。这篇文章简单介绍一下CPython的线程与GIL实现。

# 解释器状态

----

在深入了解线程之前，我们需要先了解解释器，因为这里记录了线程共享的内容。解释器被抽象成PyInterpreterState类型的结构体，结构体中记录了解释器的状态。Py_Initialize函数中就会创建并初始化一个PyInterpreterState变量。

PyInterpreterState的定义位于Include/pystate.h中，可以看到加载的module、builtin module等内容都存储在这里，所以这些内容是线程间共享的。此外，PyInterpreterState中还记录了PyThreadState的一个指针，可以推测出一个解释器下挂有多个线程这样的逻辑结构。

虽然默认情况下一个进程里面会有一个解释器，但是在需要的情况下，一个进程里面可以创建多个解释器。多个解释器是以链表的形式存在的，链表的头指针就记录在interp_head变量当中。不同的解释器有“完全独立“的Python运行环境，但是这种”完全独立“是相对的，因为依旧有一些东西会共享。所以一个进程里面会有一个GIL，而不是一个解释器有一个GIL。

# 创建线程与GIL

---

CPython将线程抽象为PyThreadState，每个线程都会对应一个PyThreadState类型的结构体。与解释器一样，Py_Initialize函数会创建一个PyThreadState变量，这个对应了进程中的第一个线程。

PyThreadState的定义位于Include/pystate.h中，可以看到每个线程都记录了一个frame以及自己所归属的解释器。多个PyThreadState以双向链表的形式存在，头指针记录在解释器单中。

Python使用threading module提供了多线程的API，但是底层的线程创建由_thread module完成，其实现在Modules/_threadmodule.c文件中。创建线程的函数如下所示：

{% highlight c %}

static PyObject *
thread_PyThread_start_new_thread(PyObject *self, PyObject *fargs)
{
    PyObject *func, *args, *keyw = NULL;
    struct bootstate *boot;
    long ident;
    ...
    boot = PyMem_NEW(struct bootstate, 1);
    ...
    boot->tstate = _PyThreadState_Prealloc(boot->interp);
    ...
    PyEval_InitThreads(); /* Start the interpreter's thread-awareness */
    ...
    ident = PyThread_start_new_thread(t_bootstrap, (void*) boot);
    ...
    return PyLong_FromLong(ident);
}   

{% endhighlight %}

其中PyEval_InitThreads完成了GIL的初始化，在只有一个线程的时候，GIL以及相关机制是未激活的。PyThread_start_new_thread的相关部分看到代码就一目了然了：

{% highlight c %}
/* Python/thread_pthread.h */

long
PyThread_start_new_thread(void (*func)(void *), void *arg)
{
    ...
    status = pthread_create(&th,
#if defined(THREAD_STACK_SIZE) || defined(PTHREAD_SYSTEM_SCHED_SUPPORTED)
                             &attrs,
#else
                             (pthread_attr_t*)NULL,
#endif
                             (void* (*)(void *))func,
                             (void *)arg
                             );
    ...
}

/* Modules/_threadmodule.c */

static void
t_bootstrap(void *boot_raw)
{
    ...
    tstate = boot->tstate;
    tstate->thread_id = PyThread_get_thread_ident();
    _PyThreadState_Init(tstate);
    PyEval_AcquireThread(tstate);
    nb_threads++;
    res = PyEval_CallObjectWithKeywords(
        boot->func, boot->args, boot->keyw);
    ...
}

{% endhighlight %}

GIL的主要实现部分在Python/ceval_gil.h文件当中，GIL自身仅仅是一个atomic_int，由一个互斥锁保护，具体内容如下所示：

{% highlight c %}

/* Python/ceval_gil.h */

#define MUTEX_T PyMUTEX_T

static _Py_atomic_int gil_locked = {-1};
static COND_T gil_cond;
static MUTEX_T gil_mutex;

/* Python/condvar.h */

#define PyMUTEX_INIT(mut)       pthread_mutex_init((mut), NULL)
#define PyMUTEX_T pthread_mutex_t

{% endhighlight %}

# 线程调度

---

虽然CPython的线程是操作系统原生线程，但是线程调度并不是完全交给操作系统的。在较早版本的CPython中，当一个线程获得GIL后，会执行一定数量的字节码，然后自己释放GIL。在CPython-3.2中，一定数量的字节码改成了一定的执行时间，一个线程执行一定的时间后会释放GIL。这样的调度逻辑被放置在PyEval_EvalFrameEx函数中：

{% highlight c %}

PyObject *
PyEval_EvalFrameEx(PyFrameObject *f, int throwflag)
{
    for (;;) {
        ...

#ifdef WITH_THREAD
            if (_Py_atomic_load_relaxed(&gil_drop_request)) {
                /* Give another thread a chance */
                if (PyThreadState_Swap(NULL) != tstate)
                    Py_FatalError("ceval: tstate mix-up");
                drop_gil(tstate);

                /* Other threads may run now */

                take_gil(tstate);

...
{% endhighlight %}

可以看到，关键的两个函数是drop_gil与take_gil，在Python/ceval_gil.h文件中定义。具体代码就不贴了，毕竟代码里面有注释，很容易看懂。

有一个值得注意的地方就是，除了gil_mutex，CPython还用了第二个锁switch_mutex来保证当前线程释放GIL后另外一个线程一定会获得GIL。如果不这样做，当前线程释放GIL又立马获得GIL是有可能的。

# 小结

---

GIL的存在使得CPython无法很好地利用多核，而GIL存在的根本原因是CPython的内存自动回收机制。Python社区曾经试图实现去除GIL版本的CPython，但是这种实现会显著降低单线程的性能，所以最后放弃了。可以认为，除非有新的内存自动回收机制，不然在可预计的时间内GIL将一直伴随着CPython。

既然GIL的存在原因是内存自动回收机制，那么在不涉及到内存分配与回收的情况下是不是就可以无视GIL了呢？答案是肯定的。在IO的时候解释器会自己释放GIL，这是CPython的默认机制。在C extension modules中进行较长时间的数值计算时也可以释放GIL，通常的做法是将相关的数据复制到内存中后释放GIL，在计算过程中不调用CPython的相关API，在计算完成后再获得GIL。在非数值计算中也可以在相关API调用之间释放GIL，但是频繁地这样做会降低性能。


(the end)
