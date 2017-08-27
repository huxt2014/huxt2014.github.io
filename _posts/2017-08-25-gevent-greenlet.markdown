---
layout: post
title:  "gevent基础（一）：基于greenlet的协程"
date:   2017-08-25 21:00:00 +0800
categories: python
tags: python
---

Python主要的并发模型中有进程、线程和协程三种，其中协程的实现方式至少有两种：一种是Python3中原生的机于generator和yield实现的协程，另外一种是Python2和Python3均适用的第三方模块greenlet。这篇文章主要探索一下greenlet的实现方式。

注：这篇文章仅涉及greenlet C代码中的主要切换逻辑，实际上greenlet切换的实现涉及到了函数栈的操作以及利用汇编对寄存器进行操作，如果想了解这部分细节，请参考：[python协程的实现（greenlet源码分析）](http://blog.csdn.net/fjslovejhl/article/details/38824963)

# 直观体验

---

greenlet的基本使用示例如下所示，代码中创建了两个greenlet实例，并且使用g1.switch()来启动第一个greenlet实例，即执行func1。

{% highlight python %}
from greenlet import greenlet

def func1():
    print("g1 begin")
    g2.switch()
    print("return to g1")
    g2.switch()


def func2():
    print("g2 begin")
    g1.switch()
    print("reenter g2")

g1 = greenlet(func1)
g2 = greenlet(func2)

g1.switch()

# 输出：
# g1 begin
# g2 begin
# return to g1
# reenter g2

{% endhighlight %}

我们先站在函数调用的角度来看一下。func1开始执行之后调用了func2，func2执行了一部分之后switch到func1了。如果这次switch是一种return，那么func2返回后就应该结束了。但是执行逻辑从func2跳到func1后，竟然还可以继续从func1跳到func2，执行func2未执行完成的部分。这个从函数调用的角度就无法理解了。

如果类比成线程，greenlet(func1)相当于Thread(func1)，第一次调用g1.switch()相当于g1.start()，那么就可以理解了：g1和g2是两个协程，与线程一样它们可以相互切换来达到和线程一样的并行效果，只不过线程间的切换由Python解释器控制，而协程间的切换完全由应用程序控制。

# greenlet实例的初始化

---

先回忆一下在Python中是如何使用线程的。main函数在MainThread当中，main函数中初始化一个线程时，需要传入一个函数，启动线程后这个函数就会被执行。在greenlet中，这种逻辑也是相似的，从以下的实现中可以看出来。

值得一提的是，greenlet在初始化的时候会设置一个parent，所以每个greenlet实例之间都是有依赖关系的，就好像一棵树一样。这棵树的根节点，也就是root greenlet在模块初始化的时候生成。当MainThread中import greenlet之后，root greenlet就生成了，而MainThread的执行情况也由root greenlet记录。

此外，不仅仅是MainThread，每个thread中都会有各自的root greenlet，都有各自的树结构。这个是由宏STATE_OK保证的，具体实现就不在这里列出来了。


{% highlight c %}

static PyObject* green_new(PyTypeObject *type, PyObject *args, PyObject *kwds)
{
    PyObject* o = PyBaseObject_Type.tp_new(type, ts_empty_tuple,
                                           ts_empty_dict);
    if (o != NULL) {
        if (!STATE_OK) {
            Py_DECREF(o);
            return NULL;
        }
        Py_INCREF(ts_current);
        /* 默认将当前的greenlet作为parrent */
        ((PyGreenlet*) o)->parent = ts_current;
    }
    return o;
}

static int green_init(PyGreenlet *self, PyObject *args, PyObject *kwargs)
{
    ...
    static char *kwlist[] = {"run", "parent", 0};
    ...
    if (run != NULL) {
        /* 保存了run */
        if (green_setrun(self, run, NULL))
            return -1;
    }
    if (nparent != NULL && nparent != Py_None)
        /* __init__ 中可以设置parent */
        return green_setparent(self, nparent, NULL);
    return 0;
}

/* 全局变量，一直会指向switch-to greenlet*/
static PyGreenlet* volatile ts_target = NULL;
/* 全局变量，一直会指向当前正在执行的greenlet*/
static PyGreenlet* volatile ts_current = NULL;

static PyGreenlet* green_create_main(void)
{
    PyGreenlet* gmain;
    PyObject* dict = PyThreadState_GetDict();
    ...
    gmain = (PyGreenlet*) PyType_GenericAlloc(&PyGreenlet_Type, 0);
    ...
    gmain->run_info = dict;
    ...
    return gmain;
}


PyMODINIT_FUNC
initgreenlet(void)
{
    /* module初始化函数，参考Python C API */
    ...
    ts_current = green_create_main();
    ...
}

{% endhighlight %}

# switch细节

---

以下摘录的关键部分代码展现了switch函数是如何实现的，具体的说明见注释。

{% highlight c %}

# 这个就是greenlet的switch方法
static PyObject* green_switch(
    PyGreenlet* self,
    PyObject* args,
    PyObject* kwargs)
{
    Py_INCREF(args);
    Py_XINCREF(kwargs);
    return single_result(g_switch(self, args, kwargs));
}


static PyObject *
g_switch(PyGreenlet* target, PyObject* args, PyObject* kwargs)
{
    ...
    /* find the real target by ignoring dead greenlets,
       and if necessary starting a greenlet. */
    while (target) {
        if (PyGreenlet_ACTIVE(target)) {
            ts_target = target;
            /* switch到某个已经启动了的greenlet会执行这个，即第N次执行
               g1.switch()、g2.switch()时执行的是这个(N>=2) */
            err = g_switchstack();
            /* 到这里时，已经切换到了target greenlet的函数栈了，
               即之后会执行target greenlet */
            break;
        }
        if (!PyGreenlet_STARTED(target)) {
            /* 某个greenlet第一次调用switch，即第一次执行g1.switch()、
               g2.switch()时执行的是这个，类似于Thread.start。
               需要注意的是，当前处于parent的函数栈中，g_initialstub
               中会切换到target，target又可能切换到其他的greenlet，所以
               g_initialstub是不会自己返回的。 */
            void* dummymarker;
            ts_target = target;
            err = g_initialstub(&dummymarker);
            if (err == 1) {
                continue; /* retry the switch */
            }
            /* target执行完成后，会主动switch到parant，这个时候才会这里。
             * 这个时候err=0 */
            break;
        }
        target = target->parent;
    }

    ...
    /* 好了，可以去执行target的函数栈了 */
}

static int GREENLET_NOINLINE(g_initialstub)(void* mark)
{

    PyGreenlet* self = ts_target;
    ...
    /* 目前还是在parant中哦，马上要保存parent的函数栈，并且即将要切换到target */
    /* perform the initial switch */
    err = g_switchstack();

    /* here，先跳过，看完后面的代码后再来看这一段。
       由于上面保存了parent的函数栈，所以下面PyEval_CallObjectWithKeywords
       执行完成后，调用g_switch(parent, result, NULL)之后，会第二次运行到这里哦。
       这个就是所谓的第二次返回了。*/

    /* returns twice!
       The 1st time with err=1: we are in the new greenlet
       The 2nd time with err=0: back in the caller's greenlet
    */
    if (err == 1) {
        ...
        /* 这里已经切换到target了 */

        if (args == NULL) {
            /* pending exception */
            result = NULL;
        } else {
            /* call g.run(*args, **kwargs) */
            /* 开始执行函数，类似于Thread.run。需要注意的是，run里面可能
               会有任意多的switch调用，如果switch到了其他greenlet，
               PyEval_CallObjectWithKeywords是不会返回的。 */
            result = PyEval_CallObjectWithKeywords(
                run, args, kwargs);
            Py_DECREF(args);
            Py_XDECREF(kwargs);
        }
        ...

        /* 某个greenlet执行完成后，即PyEval_CallObjectWithKeywords返回了，
           才会运行到这里。 相当于Thread.run执行完返回了。 */
        /* jump back to parent */
        self->stack_start = NULL;  /* dead */
        for (parent = self->parent; parent != NULL; parent = parent->parent) {
            result = g_switch(parent, result, NULL);
            /* 正常情况下并不会运行到这里，调用g_switch后，会跑到上面的 here 注释哦 */
            /* Return here means switch to parent failed,
             * in which case we throw *current* exception
             * to the next parent in chain.
             */
            assert(result == NULL);
        }
        /* We ran out of parents, cannot continue */
        PyErr_WriteUnraisable((PyObject *) self);
        Py_FatalError("greenlets cannot continue");
    }

    return err;
}


static int g_switchstack(void)
{
    /* 保存当前执行环境 */
    ...

    /* 当前的greenlet切换出去，target切换进来，汇编实现 */
    err = slp_switch();

    ...

    if (err < 0) {   /* error */
        ...
    }
    else {
        ....
        ts_current = target;
        ts_origin = origin;
        ts_target = NULL;
    }
    return err;
}    

{% endhighlight %}

# 小结

---

如果要彻底看到greenlet是如何实现的，感觉有一点烧脑。即使不完全看懂，理解了如下greenlet的主要逻辑之后，对于使用greenlet还是很有帮助的：

1. greenlet中，greenlet实例可以通过调用switch方法来切换到其他的greenlet实例。这个逻辑和线程之间的切换有点像，只不过greenlet中的切换由应用程序完全控制。
2. 每个greenlet实例都会有一个parent greenlet实例，所有的greenlet实例被组织成一个树型结构，root greenlet在import greenlet的时候自动创建。当某个greenlet实例执行完成后，会自动切换到parent greenlet实例。当所有greenlet实例都执行完了，执行逻辑会回到root greenlet，最后程序执行完成。
3. 每个thread里面都会有各自的root greenlet，都有各自的树型结构。每个greenlet实例只能属于某一个thread，属于不同线程的greenlet实例不能相互切换。

(the end)

