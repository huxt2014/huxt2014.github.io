---
layout: post
title:  "OpenID Connect：协议概述"
date:   2017-09-17 20:00:00 +0800
categories: auth
tags: auth
---

OpenID Connect是一个基于OAuth 2.0扩展得到的认证+授权协议，它基于OAuth 2.0的交互流程，让Authorization Server能够提供认证服务，并且能够提供用户的基本信息。这片文章基于[官方文档](http://openid.net/specs/openid-connect-core-1_0.html)来概括一下OpenID Connect的核心部分是怎样在OAuth 2.0上进行扩充的。

# OpenID简介

---

OpenID希望支持这样一个场景：当你在使用A系统的时候，A系统需要认证你的身份（通常通过log in来实现），但是你不想在A系统上面创建账户因为你在B系统上已经有一个账户了。于是A将认证转交B给，你在B认证通过后，B告诉A“这个人在我这认证通过了，他就是他”并把你的基础信息交给A，这样A就算你认证通过了。

简单来讲，OpenID让一个服务提供方能够有基本的用户体系，但是又不用负责管理用户名和密码，对于用户的认证可以交给第三方来进行。在这样一个场景中，第三方扮演了identify provider的角色。更近一步说，OpenID是一个支持去中心化认证（所谓的open）的协议，因为一个服务提供方可以信任多个identify provider。

在OpenID connect之前，OpenID有两个旧的版本：OpenID 1.0和OpenID 2.0。这两个版本的OpenID有或多或少的缺陷／问题，所以没有顺利地推广开。在结合OAuth 2.0后，OpenID有了第三代，也就是OpenID connect，这个版本在各个方面都有了很大改善。

# 基本流程

---

OpenID Connect是在在OAuth 2.0上扩展而得到的，主要流程两者基本上是一样的，其中扩展／修改部分列举如下：

1. OpenID connect将OAuth 2.0中的authorization server称为OpenID Provider(OP)，将OAuth 2.0中的resource server称为Relying Party (RP)。其中，OP同时负责认证与授权。
2. 认证流程有三种方式： Authorization Code Flow，Implicit Flow和Hybrid Flow（基于OAuth 2.0 Multiple Response Type Encoding）。
3. RP向OP申请认证时需要在scope中添加openid，表明是OpenID Connect协议的认证。在收到认证请求时，OP需要确认scope中含有openid，如果没有，那么可以认为对应的请求是普通的OAuth 2.0请求。
4. Authorization Endpoint需要同时完成授权和认证的工作。
5. Token Endpoint需要返回ID Token和Access Token，可能会有Refresh Token。其中，ID Token携带了用户的基本信息，由JWT签名。
6. Client获得ID token后必须进行一系列的校验，例如加密与解密、iss与OpenID Provider相匹配、aud与client_id相匹配等等。
7. ID Tokenz中的详细内容被称为claim，规范中提供了一系列标准的claim，在具体实现中也可以添加非标准claim。
8. 新增UserInfo Endpoint，用于用ID Token查询更详细的用户信息。在查询中可以查询全部的用户信息，也可以制定查询某几个claim。
9. RP向OP发送认证请求时，该请求的全部内容可以被封装在一个参数当中，该封装方式参考Request Object部分的说明。
10. 其他内容参考规范。

# ID Token

---

ID Token是OpenID Connect中添加的一个关键性内容，它在用户于OP完成认证后由OP生成，之后被传递给RP，其中携带了用户的信息。ID Token中携带的信息被称为claim，其中关键性的内容包括但不仅限于:

* iss，Issuer Identifier，通常是OP的https开头的URL。
* sub，end-user identifier，也就是用户在OP中的ID，这个ID在OP处一经生成就不应该再发生改变。
* aud，也就是client_id。
* exp，过期时间。

规范要求ID Token必须有JWS签名，也可以先经JWS签名然后经JWE加密，相关的密钥需要OP在jwks_uri发布。


(the end)
