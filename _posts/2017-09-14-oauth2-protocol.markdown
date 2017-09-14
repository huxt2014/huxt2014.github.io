---
layout: post
title:  "OAuth2.0：协议概述"
date:   2017-09-14 21:00:00 +0800
categories: auth
tags: auth
---

OAuth协议是一个用于第三方授权的协议，其为在不提供认证凭据（用户名密码等）的情况下授权给第三方访问指定的资源提供了一个解决方案。OAuth协议有1.0和2.0两个版本，2.0不兼容1.0。这篇文章基于[rfc6749](https://tools.ietf.org/html/rfc6749)与[rfc6750](https://tools.ietf.org/html/rfc6750)简单介绍一下OAuth2.0协议的授权流程和应用场景。

# 基本原理

---

你在使用某个APP的时候，这个APP希望访问你github账户的某些非公开信息，你也打算允许它这样做。既然是非公开信息，那么github显然不能随意给其他人。在传统的方法中，你需要将你github账户的认证凭据交给APP，APP利用你的凭据在github认证后，就可以获得你的账户信息了。但是，这样做有一系列问题，例如APP拥有与你一样的权限，也就是所有权限，你不知道它还会不会做其他事情；你不知道APP会不会把你的登录凭据给其他人。

以上场景中存在在的问题来源于传统授权的实现方式：**认证即授权**。具体来讲，当你利用认证凭据（最常见的就是用户名和密码）在服务端进行认证后（最常见的就是登录），你就获得了对应的权限；当你需要获得某些权限，那么你必须进行认证：这两者是无法分割的。

OAuth提供了一个新的机制，将认证与授权分割开来。在新的机制下，当这个APP希望访问你github的非公开信息，它会将你重定向至github的授权界面，你在github完成认证与授权操作后，github会生成一个token并交给APP，之后APP就可以凭这个token访问你的账户信息了。

在OAuth提供的场景中有四个主体：你（user/resource owner），APP（client），github（resource server），github的授权界面（authorization server）。在这样一个交互过程中，认证凭据和授权凭据被分割开了，client可以仅仅持有授权凭据来访问resource server，而OAuth就是为这样一个交互过程提供了一个规范。

以上的交互过程可以由下图概括，即client在resource owner的协助下获得authorization grant，之后client凭借authorization grant换取access token，最终使用access token来访问资源。图中仅仅描绘了一个大概流程，在具体实现中OAuth提供了四种可选方案。

{% highlight text %}
     +--------+                               +---------------+
     |        |--(A)- Authorization Request ->|   Resource    |
     |        |                               |     Owner     |
     |        |<-(B)-- Authorization Grant ---|               |
     |        |                               +---------------+
     |        |
     |        |                               +---------------+
     |        |--(C)-- Authorization Grant -->| Authorization |
     | Client |                               |     Server    |
     |        |<-(D)----- Access Token -------|               |
     |        |                               +---------------+
     |        |
     |        |                               +---------------+
     |        |--(E)----- Access Token ------>|    Resource   |
     |        |                               |     Server    |
     |        |<-(F)--- Protected Resource ---|               |
     +--------+                               +---------------+

{% endhighlight %}

# 四种方案

---

#### 1. Authorization Code

client将user重定向至authorization server，user在authorization server完成授权获得authorization code。之后authorization server将user重定向至client，此过程会携带authorization code。client在authorization server将authorization code替换成access token。

#### 2. Implicit

client在user的帮助下直接从authorization server获得token，而不是获得authorization code后再去换取token。简化版的authorization code流程，适用于用脚本语言（例如js）在浏览器中实现的clients。

#### 3. Resource Owner Password Credentials

client直接持有用户的认证凭据，然后使用用户的认证凭据直接作为authorization code换取access token。这种方式仅适用于可信赖的client（不会窃取用户凭据）。

#### 4. Client Credentials

client拥有自己的凭据，使用自己的凭据（而不是用户的凭据）作为authorization code换取access token。这种方式适用于资源本身就归client所有，或者在资源上对应实施了某种额外的访问策略。

# Client相关规范

---

#### 1. 分类

根据能否有效地控制client credentials和token（不会泄密），client可以分为两类： confidential和public。一般来讲，web应用属于confidential类型，因为client credentials和token都存储于服务器；user-agent-based application和native application一般属于public类型。

如果client是confidential类型的，那么client与authorization server之间应该存在一个认证机制，让authorization server可以确认某个client的identifier。具体的认证机制可以视情况而定，例如password，public／private key pair。

#### 2. 注册

设想一个使用场景：用户在授权若干个APP访问自己的github账户后，他在自己github账户的相关界面上应该可以查看到自己到底给哪些APP授权了，并且应该可以有选择性地取消某些APP的授权。此外，当github授权界面希望将用户重定向至APP的时候，它应该知道到底重定向至哪个URI。为了满足这些要求，client在使用OAuth协议之前，应该有一个在Authorization server注册的过程。

为了完成client的注册，client需要提供自己的类型（confidential or public）、重定向URI以及其他authorization server要求的信息。相应的，authorization server也应该为client分配一个identifier，用户可以通过这个来确认自己给什么client分配了权限。

client可以通过各种方式完成注册，例如HTML表单、证书、authorization server利用可信通道自主发现。

# token相关规范

---

#### 1. access token的格式与validition

access token承载了授权信息，每一个token应该有访问权限、有效期等信息，但是作为一个相对抽象的对象，OAuth并没有做更具体的规定。规范中说明access token可以有如下两种实现策略：

1. access token仅仅作为一个检索依据，其本身不携带访问权限、有效期等信息，resource server需要根据access token去其他地方（通常是authorization server）查询。
2. access token本身就携带了访问权限、有效期等信息，resource server能够仅仅通过access token就完成validation。

所以，access token的格式以及resource server如何做validation很大程度上由具体实现来确定。不过，对于第一种策略，OAuth给出了一个示例：[bearer token](https://tools.ietf.org/html/rfc6750)。

#### 2. bearer token的传输方式

access token通常以字符串的形式存在，在client访问resource server的时候需要携带这个凭据。携带的方式可以有如下三种：

1. Authorization Request Header Field：放在header的Authorization字段中。
2. Form-Encoded Body Parameter：放在request-body中，以access_token标示。
3. URI Query Parameter：放在RRI中。

需要注意的是，OAuth规定涉及到access token传输的交互必须用TLS……

#### 3. access scope

OAuth规定通过scope字段来确定授权范围，也就是说，每个access token都有一个scope属性，通过scope来确定可以访问哪些资源，不可以访问哪些资源。scope字段的内容是个大小写敏感的字符串，字符串中能够用空格分割出多个小段。至于字段内容与权限如何对应，OAuth并没有给出规定，由具体实现来确定。

#### 4. Refresh token

在上述流程中，client利用authorization grant可以换取token。对于这个换取的操作，OAuth还提供了一个可选方案，即利用authorization grant可以换取refresh token+token，其中，refresh token可以在token失效的时候用来获得新的token。这样的流程如下图所示：

{% highlight text %}
  +--------+                                           +---------------+
  |        |--(A)------- Authorization Grant --------->|               |
  |        |                                           |               |
  |        |<-(B)----------- Access Token -------------|               |
  |        |               & Refresh Token             |               |
  |        |                                           |               |
  |        |                            +----------+   |               |
  |        |--(C)---- Access Token ---->|          |   |               |
  |        |                            |          |   |               |
  |        |<-(D)- Protected Resource --| Resource |   | Authorization |
  | Client |                            |  Server  |   |     Server    |
  |        |--(E)---- Access Token ---->|          |   |               |
  |        |                            |          |   |               |
  |        |<-(F)- Invalid Token Error -|          |   |               |
  |        |                            +----------+   |               |
  |        |                                           |               |
  |        |--(G)----------- Refresh Token ----------->|               |
  |        |                                           |               |
  |        |<-(H)----------- Access Token -------------|               |
  +--------+           & Optional Refresh Token        +---------------+

{% endhighlight %}

除了token失效可以利用refresh token更新外，如果希望更改权限范围，也可以通过重新获得token来实现。由此可以看出，在一个refresh token的生命周期当中可能有多个access token，refresh token仅用于与authorization server的交互而不会用于与resource server的交互。

# 应用

---

OAuth 2.0可以有如下几种应用：

1. 第三方授权。这也是OAuth提出的初衷，前文已经描述了对应的应用场景，这里就不赘述了。
2. 伪认证。既然传统的登录方式是“认证即授权”，那么自然也可以反过来：“授权即认证”。在这种场景中有一个问题，那就是即使得到授权了，resource owner还是不知道user到底是谁。
3. 认证+授权。OAuth2.0顶多做到伪认证，因为它没有规定用户的identity怎么确认，如果能够填补上这一块就OK了。OpenID Connect在OAuth2.0的基础上加了一层identity的逻辑，让resource owner在拿到授权凭证的同时也拿到用户信息，这样就能同时完成认证+授权了。
4. 单点登录（single sign-on，SSO）。SSO解决的问题是用户在访问某个服务时登录了一次，访问其他服务就不需要再登录了，这个时候多个服务共用一个授权中心。要实现SSO，本质上需要维护一个global session以及各个服务对应的client session，Central Authentication Service利用CAS协议来达到这个目的。虽然OpenID Connect是一个去中心化的认证方案，但是这不妨碍它用于中心化的认证，所以Keycloak用OpenID来实现了SSO。

# 其他

---

1. OAuth 2.0交互流程的规范仅仅是为HTTP协议制定的，在他协议下的使用就看大家各自发挥了。
2. OAuth 2.0 的client可以是web applications, desktop applications, mobile phones, 或者living room devices. 
3. OAuth 2.0 规范并没有规定签名、加密等内容，这些内容依赖与TLS.
4. OAuth 2.0没有规定resource server与authorization server之间的交互方式。它们可以是一个整体，也可以是相互独立的服务。一个authorization server可以给多个resource server授权。


(the end)
