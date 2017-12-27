---
layout: post
title:  "HTTP协议：概述"
date:   2017-12-27 21:00:00 +0800
categories: web
tags: web
---

HTTP协议是一个应用层协议，用于传输hypertext／hypermedia information，可以认为HTTP协议是Web应用的基础。这篇文章希望从相关RFC中抽出一些关键的信息，从而帮助从较高层次理解HTTP协议，同时为其他细节的内容起到索引的作用。

#### 标准化工作

---

HTTP协议的标准化工作由Internet Engineering Task Force (IETF)和the World Wide Web Consortium (W3C)主导，并由RFC的方式发布。在标准化的过程中，存在多个版本的HTTP协议，但是目前最常见的是HTTP／1.1和HTTP／2.0两个版本。HTTP／1.1相关的比较关键的RFC为1999年发布的[RFC 2616](https://tools.ietf.org/html/rfc2616)，以及2014年修订后得到的一组RFC：

* [RFC 7230: Message Syntax and Routing](https://tools.ietf.org/html/rfc7230)
* [RFC 7231: Semantics and Content](https://tools.ietf.org/html/rfc7231)
* [RFC 7232: Conditional Requests](https://tools.ietf.org/html/rfc7232)
* [RFC 7233: Range Requests](https://tools.ietf.org/html/rfc7233)
* [RFC 7234: Caching](https://tools.ietf.org/html/rfc7234)
* [RFC 7235: Authentication](https://tools.ietf.org/html/rfc7235)

HTTP／2.0于2015年发布为[RFC 7540](https://tools.ietf.org/html/rfc7540)。

#### 参与HTTP通信的各类角色

---

HTTP消息的发送和接收由程序完成，在HTTP通信中这些程序扮演了不同的角色，RFC正是对这些程序／角色作出了规范。这些程序／角色为：server、client、sender、recipient、user agent、origin server、intermediary、proxy、gateway、cache。

client与server是指一个连接两端的程序所扮演的角色。例如程序A直接向程序B发送请求，B给出回应，那么在这个连接中，A是client，B是server。A有可能同时是client与server，比如A给B发送请求，同时又能够给C回应。sender和recipient有差不多的意思。

如果一个client能够主动发送HTTP请求，那么这个client可以称为user agent（UA）；如果一个server对HTTP请求能够主动地给出回复，那么这个server可以称为origin server。所以，user agent和origin server也是一个连接两端的程序所扮演的角色，一个程序有可能同时是user agent和origin server。

在user agent与origin server之间，可以有intermediaries。intermediary可以认为是一个程序，而不是角色。根据功能和作用，常见的intermediary可以分为proxy、gateway和tunnel。当然，一个intermediary可以同时实现proxy和gateway的功能。

对于proxy，RFC对其有单独的规范，user agent应该能够根据配置信息知道连接的另外一端是不是proxy。对于gateway，当其接受HTTP请求和返回响应时，可以认为是origin server，遵循origin server的规范；当其发送HTTP请求时，可以认为是user agent，遵循user agent规范。对于tunnel，由于它只起一个通道的作用，可以认为不扮演任何角色，不是client也不是server。

任何client和server都可以有cache，所谓的cache就是一个与client或者与server在一起的、能够存储信息的系统。由于tunnel不是client，也不是server，所以tunnel不具备cache。

#### URI与资源标识

---

client使用URI来标识请求的资源，URI的组成部分如下所示：

{% highlight text %}

   foo://example.com:8042/over/there?name=ferret#nose
   \_/   \______________/\_________/ \_________/ \__/
    |           |            |            |        |
 scheme     authority       path        query   fragment

{% endhighlight %}

关于URI还有一些需要注意的地方：

* authority中可能还会有user information，完整格式为：[ userinfo "@" ] host [ ":" port ]
* 如果在构建URI时指定了端口，且该端口与scheme的默认端口相同，那么最终生成的URI中应该省略端口。
* absolute URI指从scheme到query的部分，仅去掉fragment。

URI是client在HTTP通信中用来标识资源的方式，但是在整个HTTP通信中这不是唯一的方式：在HTTP message中使用request target来标示，在origin server需要从HTTP message中恢复出effective request URI。

#### HTTP message

---

HTTP message格式相关的规范如下所示：
{% highlight text %}
HTTP-message   = start-line
                 *( header-field CRLF )
                 CRLF
                 [ message-body ]


start-line     = request-line / status-line

request-line   = method SP request-target SP HTTP-version CRLF

status-line = HTTP-version SP status-code SP reason-phrase CRLF

header-field   = field-name ":" OWS field-value OWS

message-body = *OCTET

{% endhighlight %}

HTTP请求和HTTP响应不同的地方仅在于start-line。


#### REST风格

---

2000年，Roy Thomas Fielding的论文 Architectural Styles and the Design of Network-based Software Architectures腾空出世，提出了REST风格的构架。之后，各种REST的最佳实践、支持REST风格的框架如雨后春笋。其实，正如论文里面所说，REST风格是对构架的比较高层次的抽象，具体落地的过程就仁者见仁了。不过，REST风格是作者在修订HTTP规范时归纳出来的，同时这个风格也被融入到HTTP规范当中。所以，HTTP规范在某种程度上是REST风格更具体的阐述。

HTTP协议是一个无状态（stateless）的协议。所谓的无状态是指每次request-response交互都是一个完整的过程，多次request-response之间没有依赖，每个request／response都能够完整地表达自身。因此，对于来自同一个连接的若干个请求，server端不应该推测最末端的client是否是同一个。这种stateless的特性属于HTTP协议自身的特点，并不妨碍将HTTP协议应用于一些有状态的场景，例如保持登陆状态、某个业务需要按照若干步依次完成。

HTTP协议是一个抽象的协议，client通过URI来请求资源时，得到的仅仅是资源的representation，不需要也不应该关心资源在server端是以什么形式存在的，也不需要知道server端的实现细节。同样，server端仅仅需要按照request返回信息即可，不需要也不应该关心client的实现细节。

#### Cookie

---

Cookie相关的规范并不属于HTTP的规范，Cookie仅涉及到如何使用某几个HTTP header来达到某些目的。历史上存在过三个Cookie的规范，分别为Netscape cookie specification、[RFC 2109](https://tools.ietf.org/html/rfc2109)、[RFC 2965](https://tools.ietf.org/html/rfc2965)，不过实践上大家都没有遵循这些规范。目前有参考价值的规范为[RFC 2965](https://tools.ietf.org/html/rfc2965)，这个规范可以说是对目前公认实践方式的归纳，以及一些建议。

(the end)

