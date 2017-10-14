---
layout: post
title:  "python解释器：执行流程概述"
date:   2017-10-14 21:00:00 +0800
categories: python
tags: python
---

Python是一种动态的解释型语言，在运行Python程序时，python解释器会逐条读取指令然后做出相对应的动作。虽然这种方式效率偏低，但是提高了灵活性。这篇文章基于Python-3.5.3的源码，以python解释器的程序入口为起点，简要介绍一下python解释器的初始化以及四种运行模式，为后续内容做好铺垫。

# 程序入口

---

下载并解压缩源码后，可以发现python解释器的入口位于Programs/python.c文件中。这个文件中的main函数内容不是很多，只是做了一些简单工作后便转入执行Py_Main，相当于是对Py_Main函数进行了一个简单包装，所以可以认为解释器实现的主要部分在Py_Main中。根据官方文档对Py_Main的解释可以做出确认：

> int Py_Main(int argc, wchar_t **argv)
> 
> The main program for the standard interpreter. This is made available for programs which embed Python. The argc and argv parameters should be prepared exactly as those which are passed to a C program’s main() function.

进一步说，没有Py_Main的话，我们只能从python可执行文件来运行解释器，进而运行Python程序；有了Py_Main后，就可以从C的层面来运行解释器了。

Py_Main的实现部分位于Modules/main.c，它主要的功能是掉用Py_Initialize函数初始化解释器，然后执行Python代码。

# 四种运行模式

---

正如Python官方文档所说，Py_Main是一个very high level layer的API，通过它可以以四种不同的模式运行python解释器：执行字符串、调用module、交互模式以及运行程序。

#### 1. 执行字符串

执行字符串就是将Python代码／程序以字符串的方式喂给解释器，这种模式中执行逻辑从Py_Main进入经过PyRun_StringFlags、PyParser_ASTFromStringObject、PyAST_CompileObject后最终会落到PyEval_EvalCode中。

PyEval_EvalCode函数有三个参数，第一个参数是字节码，由Python代码编译得到，第二、三个参数分别是globals和locals，也就是global、local作用域／scope。需要注意的是，globals和locals是从__main__ module拿到的。

#### 2. 调用module

Python的module可以当作一个程序来执行，这种模式中执行逻辑从Py_Main进入经过runpy._run_module_as_main、exec后最终会落到PyEval_EvalCode。

传入PyEval_EvalCode的第一个参数是module对应的字节码，第二、三个参数从__main__ module获得。

#### 3. 交互模式

交互模式下，执行逻辑从Py_Main进入后最终到达PyRun_InteractiveLoopFlags中的for循环，for循环不断地从输入流中读取Python代码，编译得到字节后调用PyEval_EvalCode，globals和locals从__main__ module拿到。

#### 4. 运行程序

运行程序是最常用的模式了，这种模式中执行逻辑从Py_Main进入后到达run_pyc_file函数，最后还是调用PyEval_EvalCode函数，传入的参数是pyc文件中存储的字节码以及从__main__ module拿到的globals和locals。

# 解释器初始化

---

Py_Main函数中仅仅提供了C层面的执行Python代码的接口，并没有提供额外的功能。Python提供的C API那么多，能不能利用起来对解释器进行更深程度的介入呢。答案当然是可以，不过在那之前需要调用Py_Initialize函数初始化解释器，正如官方文档中所说：

> void Py_Initialize()
>
> Initialize the Python interpreter. In an application embedding Python, this should be called before using any other Python/C API functions; with the exception of Py_SetProgramName(), Py_SetPythonHome() and Py_SetPath().

其实，Py_Main函数在编译／执行Python代码之前也会调用Py_Initialize函数对解释器进行初始化。大概浏览一下Py_Initialize函数的实现，里面做的工作还是蛮多的，主要包括以下几部分：

1. 初始化解释器状态（记录在类型为PyInterpreterState的全局变量中）与线程状态（记录在类型为PyThreadState的全局变量中）
2. 初始话类型系统、内置的module、内置异常
3. 初始化	signal handler、standard io
4. 初始化__main__ module
5. 其他初始化

在Java的世界中，Java代码是由jvm执行的，对于jvm也相关规范介绍其结构。Python解释器和jvm是同一层次的东西，但是之前没有找到官方文档介绍Python解释器的结构，貌似在Python的世界中解释器只有实现而没有规范？如果确实是这样，那么至少可以从Py_Initialize函数中看看CPython是如何实现解释器的。

# 小结

---

从解释器的不同执行模式可以看到，最后执行逻辑都汇集到了PyEval_EvalCode函数。PyEval_EvalCode会执行Python代码编译后得到的字节码，与之相关的是local和global作用域。可以认为，PyEval_EvalCode函数就是Python程序由静态转向动态的分割点。

回顾一下C程序，当我们讨论一个C程序的静态内容时，讨论对象是编译链接后得到的可执行程序，这个文件中有机器码、各种绝对／相对地址、静态数据等等；当我们讨论C程序的运行态时，讨论对象是heap／stack中的数据、stack的调用关系等等。对比一下，这些东西在python解释器中都有与之对应的东西。

在python解释器中，Python程序的静态内容就是编译之后的字节码，字节码被抽象成PyCodeObject对象，包含了Python程序的所有内容；Python程序的动态内容包含了比较多的东西，包括解释器状态、线程状态、\_\_main\_\_ module，LEGB作用域、PyFrameObject等等，其中PyFrameObject对应了C程序中的stack。这些内容会在之后的文章中聊到。

(the end)
