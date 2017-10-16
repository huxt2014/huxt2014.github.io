---
layout: post
title:  "CPython：PyCodeObject与PyFrameObject"
date:   2017-10-16 21:00:00 +0800
categories: python
tags: python
---

python解释器（CPython）在执行Python代码之前，会先对源码进行编译，将编译结果存储在PyCodeObject对象中。在执行代码的时候，则由PyFrameObject记录运行时态的信息。这片文章主要介绍一下PyCodeObject与PyFrameObject，以及执行字节码的主要部分。

# PyCodeObject

---

PyCodeObject的定义在Include/code.h文件中，可以看到里面记录了很多信息。除了查看C代码，在Python的层面也可以查看PyCodeObject的信息，如下所示：
{% highlight python %}
In [1]: source = open('codeobject.py').read()

In [2]: print(source)

a = 1
b = 2

def test(a, c=None):
    a = 1
    c = 2

class Test:
    a = 1

def main():
    test()

if __name__ == "__main__":
    main()


In [3]: co = compile(source, 'objectcode.py', 'exec')

In [4]: dir(co)
Out[4]:
[...
 'co_argcount',
 'co_cellvars',
 'co_code',
 'co_consts',
 'co_filename',
 'co_firstlineno',
 'co_flags',
 'co_freevars',
 'co_kwonlyargcount',
 'co_lnotab',
 'co_name',
 'co_names',
 'co_nlocals',
 'co_stacksize',
 'co_varnames']

{% endhighlight %}

任何Python代码编译过后都会得到一个PyCodeObject对象。值得一提的是，当一个Python文件以import module的方式加载过后，会生成一个pyc文件，这个文件的主要内容就是一个PyCodeObject对象。说是说一个PyCodeObject对象，但是PyCodeObject对象中可以嵌套PyCodeObject对象（存储在co_consts中），也就是说PyCodeObject对象是一个嵌套结构。

PyCodeObject中除了包含字节码（存储在co_code中），还存储了另外两个比较重要的东西：名称和常量。在C程序中，变量名、函数名以及常量经过编译后都会转化为地址，在Python 中，这些信息都存储在PyCodeObject中，如下所示：

{% highlight python %}
In [5]: co.co_names
Out[5]: ('a', 'b', 'test', 'Test', 'main', '__name__')

In [6]: co.co_consts
Out[6]:
(1,
 2,
 None,
 <code object test at 0x103d16f60, file "objectcode.py", line 5>,
 'test',
 <code object Test at 0x103d16c90, file "objectcode.py", line 9>,
 'Test',
 <code object main at 0x103df4030, file "objectcode.py", line 12>,
 'main',
 '__main__')
{% endhighlight %}

可以看到，源码中定义的函数和类对应了一个PyCodeObject对象。一般来讲，一个新的命名空间／作用域时会生成一个PyCodeObject对象。

# 字节码

---

Python字节码和机器码类似，是执行逻辑的主要载体。字节码存储在co_code中，直接看的话只是一串二进制的字符串，但是Python提供了接口方便查看：

{% highlight python %}
In [13]: co.co_code
Out[13]: b'd\x00\x00Z\x00\x00d\x01\x00Z\x01\x00d\x02\x00d\x03\x00d\x04\x00\x84\x01\x00Z\x02\x00Gd\x05\x00d\x06\x00\x84\x00\x00d\x06\x00\x83\x02\x00Z\x03\x00d\x07\x00d\x08\x00\x84\x00\x00Z\x04\x00e\x05\x00d\t\x00k\x02\x00rM\x00e\x04\x00\x83\x00\x00\x01d\x02\x00S'

In [14]: from dis import dis

In [15]: dis(co)
  2           0 LOAD_CONST               0 (1)
              3 STORE_NAME               0 (a)

  3           6 LOAD_CONST               1 (2)
              9 STORE_NAME               1 (b)

  5          12 LOAD_CONST               2 (None)
             15 LOAD_CONST               3 (<code object test at 0x103d16f60, file "objectcode.py", line 5>)
             18 LOAD_CONST               4 ('test')
             21 MAKE_FUNCTION            1
             24 STORE_NAME               2 (test)

  9          27 LOAD_BUILD_CLASS
             28 LOAD_CONST               5 (<code object Test at 0x103d16c90, file "objectcode.py", line 9>)
             31 LOAD_CONST               6 ('Test')
             34 MAKE_FUNCTION            0
             37 LOAD_CONST               6 ('Test')
             40 CALL_FUNCTION            2 (2 positional, 0 keyword pair)
             43 STORE_NAME               3 (Test)

 12          46 LOAD_CONST               7 (<code object main at 0x103df4030, file "objectcode.py", line 12>)
             49 LOAD_CONST               8 ('main')
             52 MAKE_FUNCTION            0
             55 STORE_NAME               4 (main)

 15          58 LOAD_NAME                5 (__name__)
             61 LOAD_CONST               9 ('__main__')
             64 COMPARE_OP               2 (==)
             67 POP_JUMP_IF_FALSE       77

 16          70 LOAD_NAME                4 (main)
             73 CALL_FUNCTION            0 (0 positional, 0 keyword pair)
             76 POP_TOP
        >>   77 LOAD_CONST               2 (None)
             80 RETURN_VALUE
{% endhighlight %}

# PyFrameObject

---

在上一片文章中有提到，字节码最终会在PyEval_EvalCode函数中被执行，PyEval_EvalCode函数的主要部分就是创建PyFrameObject对象并为其设置好相关参数，最终掉用PyEval_EvalFrame。

PyFrameObject的定义在Include/frameobject.h文件中。其中，f_back指向前一个frame，可以看出有一个链式结构；f_code存储了PyCodeObject，这里可以拿到字节码和静态的数据；f_builtins、f_globals、f_locals都是字典，Python代码执行过程中紧密相关的LEGB中的三个即源于这里。其他更多的细节就不说了，在Python层面可以通过sys._getframe得到当前frame。

PyEval_EvalFrame的定义在Python/ceval.c当中，茫茫长的代码当中，主要部分就是不断读取字节码，通过switch函数来确定对应的动作，如下所示：

{% highlight C %}
/* Interpreter main loop */

PyObject *
PyEval_EvalFrame(PyFrameObject *f) {
    return PyEval_EvalFrameEx(f, 0);
}

PyObject *
PyEval_EvalFrameEx(PyFrameObject *f, int throwflag)
{
    PyThreadState *tstate = PyThreadState_GET();
    ...
    tstate->frame = f;
    ...

    for (;;) {
        ...
        opcode = NEXTOP();
        ....
        switch (opcode) {
            TARGET(NOP)
            	FAST_DISPATCH();
            TARGET(LOAD_FAST) {
            ...
        }
    }

}


{% endhighlight %}

# 小结

---

在日常的Python使用当中，经常会碰到一些难以从官方文档查证的东西，例如某个操作是不是线程安全的、MRO到底是如何确定的、如何在C层面debug等等。了解到以上内容后，我们就可以快速地从Python代码定位到字节码，最后再定位到具体的C实现，从而找到问题答案了。

(the end)
