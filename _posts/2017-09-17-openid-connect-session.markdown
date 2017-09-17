---
layout: post
title:  "OpenID Connect：session管理"
date:   2017-09-17 21:00:00 +0800
categories: auth
tags: auth
---

[OpenID Connect Core 1.0](http://openid.net/specs/openid-connect-core-1_0.html)中说明了OpenID Connect是如何扩展OAuth 2.0提供认证+授权功能的，这部分的介绍在[上一篇文章]({{site.baseurl}}{% post_url 2017-09-17-openid-connect-core %})中有介绍。一般情况下，认证+授权通过登录（log-on）来实现，既然有登录那么也对应有退出（log-out）的操作，也就是说需要有session管理。对于session部分，OpenID通过三篇规范来说明如何实现：[OpenID Connect Session Management 1.0](http://openid.net/specs/openid-connect-session-1_0.html)、[OpenID Connect Front-Channel Logout 1.0](http://openid.net/specs/openid-connect-frontchannel-1_0.html)、[OpenID Connect Back-Channel Logout 1.0](http://openid.net/specs/openid-connect-backchannel-1_0.html)。这篇文章基于上述三篇规范来对OpenID Connect的session管理做一个小结。

# 概述

---

加入session管理后，用户在OP完成登录的时候会拿到一个session id，标志着一个sessio开始了。这个session实质上是一个全局的session，仅由OP创建和销毁，下文中暂且称为OP session。当用户访问RP的时候，可以与RP之间再次创建session，这种暂且称之为RP session。在同时与不同RP交互的时候，RP session属于各自的RP，而OP session是在不同RP之间共享的。OpenID Connect对session的规范仅适用于OP session，RP session如何实现由RP各自决定。

OpenID Connect对session的规范适用于在这样的场景中解决log-out的问题：用户认证通过后，同时访问多个RP的时候，如果在某个RP作出了log-out操作，如何通知OP、如何确定是否要终止OP session；如果要终止OP session，是否要通知其他RP；如果要通知，应该采取什么样的方式通知。简单而言，就是如何协调OP session与RP session的销毁。

# OP session的创建与销毁

---

如果OP支持session，那么用户在OP完成认证后，除了拿到ID Token和Access Token外，还会拿到一个session state，这个就相当于是session id了。session state由指定方法计算得到，除了由OP提供，还应该能够在client 
agent（浏览器）根据指定方法计算得到。

当用户在RP退出登录的时候，在RP完成相关操作后（销毁RP session），需要被重定向至end_session_endpoint，在这里OP需要询问用户是否要在OP退出登录（销毁OP session）。

# RP session感知OP session的销毁

---

RP session与OP session之间的创建顺序是很好理解的，但是如果用户在同时使用多个RP，与各个RP之间有RP session，那么某个RP退出登录后通知OP结束OP session，其他RP如何得知这一事件并作出相应呢。对于RP如何感知OP session结束，OpenID Connect给出了三种实现方案。

#### 1. polling

轮询的方式是指在浏览器中，每个RP页面中有脚本每隔一段时间去查询浏览器中OP session的状态（一般在cookies中），如果发现OP session发生了改变，那么就可以做出反应了。这种轮询不需要去访问OP，仅需要在浏览器端进行就可以了，需要利用到postMessage方法。

在浏览器中，一般情况下属于不同origin的脚本是不能够相互访问对方的相关内容的，包括cookies。所谓的origin，是指scheme+host+port，例如https://localhost:8080就是一个origin。OpenID Connect的登录操作是在OP的域名下进行的，RP与OP之间、RP与RP之间应该允许属于不同的域名，那么RP页面中的脚本如何访问到OP域名中的cookies呢。

在轮询的方案中，RP页面需要加载一个RP iframe和一个OP iframe，其中RP iframe由RP实现，而OP iframe由RP访问OP的check_session_iframe endpoint得到。由于OP iframe是从OP的域名下拿到的，那么自然可以访问到浏览器上OP域名中cookies之类的内容，也就是可以访问到OP session的状态了。RP iframe通过postMessage方法对OP iframe轮询，让OP iframe返回OP session的状态有没有变化。

这种方案的局限性应该在于仅适用于浏览器，且不能跨多个浏览器进行。

#### 2. front-channel

在这种方式中，每个RP告诉OP自己log-out的url（一般是完整的url，不是相对路径），当用户在某个RP进行退出操作时，该RP会将用户重定向到OP的log-out页面中，OP在log-out界面中可以通过iframe嵌入其他RP的log-out url，这样用户在访问OP log-out界面的时候，就可以触发请求去访问其他RP的log-out url，达到在其他RP退出登录的目的。

这种方式局限性应该是三种方式中最小的，因为在一段不长的时间内，用户端应该可以访问所有RP。除非用户在短时间内发生异地操作使得某些RP在用户这一端无法访问使得log-out失败，不然即使是长时间间隔的异地操作，RP sesison也会因为超时而失效。

#### 3. back-channel

在前两种方式中，RP感知OP session失效都是在用户这一端通过浏览器完成的，在back-channel方式中需要OP向RP主动推送消息。具体来讲，每个RP在OP端注册log-out url，当某个RP终止了OP session的时候，OP会主动向其他RP的log-out url发送请求。

这种方式的局限性在于，当RP端有访问限制使得OP无法访问RP的log-out url，这个时候通知就失效了。

# 小结

---

OpenID Connect中给出了OP session的创建与销毁步骤，当OP session销毁的时候有三种方式可以通知到各个RP，这三种方式可以并存。另外，貌似关于session的规范还是处于draft状态。

对于session的操作，其实已经可以用于单点登录和单点退出了。


(the end)
