---
layout: post
title:  "socket的一些边界场景"
date:   2018-03-23 21:00:00 +0800
categories: web
tags: web
---

#### 1. 调用close之后内核中未发送完的数据可能会被丢弃，进而导致数据发送失败。

close之后内核如何处理未发送完的数据在一定程度上是可控的，这个控制通过SO_LINGER来设置。应用程序调用close时如果内核有数据未发送，那么可能出现以下三种场景：1）立刻返回，内核在后台将数据发送完。2）block，直到内核将数据发送完或者超时。3）立刻返回，内核可能将数据丢弃，并且发送RST。

BSD和POSIX一般都支持SO_LINGER，但是具体的实现方式需要参考对应操作系统的文档（或者源码）。

#### 2. UDP中发生Connection refused的场景解析。

UDP不像TCP那样有三次握手的过程，无法通过发起连接来事先感知对方端口是否打开，所以只能通过其他方式来判断，这个方式就是ICMP。发送端直接发送信息后，如果对方端口未打开，那么对方会回一个ICMP包，发送端内核收到ICMP包后会进行适当的处理。

需要注意的是，内核收到ICMP包后不一定会将相关信息反馈给应用层。应用层只有对socket调用了connect，或者设置了IP_RECVERR，才能获得这个信息，否则无法获得。

#### 3. 发送、接收数据时的timeout问题

Timeout的问题可以从两个层面来看，第一个是系统调用的层面，第二个是语言的层面。

从系统调用的层面来说，Linux的socket支持SO_RCVTIMEO和SO_SNDTIMEO，通过这个两个设置可以让系统调用在底层处理timeout问题，应用层仅需要关心系统调用的返回结果就行。

从语言的层面来说，以Python为例，Python的socket有settimeout这个接口，但其实现方式是在C层面用epoll/select来实现的。这种方式算虽然从Python代码的层面没有额外的函数调用，但是依旧算是在应用层解决timeout问题。


(the end)

