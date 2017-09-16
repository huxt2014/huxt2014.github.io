---
layout: post
title:  "OAuth2.0：authorization server与resource server实现"
date:   2017-09-16 21:00:00 +0800
categories: auth
tags: auth
---

上两篇文章分别介绍了[OAuth 2.0协议]({{site.baseurl}}{% post_url 2017-09-14-oauth2-protocol %})以及[OAuth 2.0 client的实现方式]({{site.baseurl}}{% post_url 2017-09-15-oauth2-client %})，熟悉这两块内容之后对OAuth 2.0就会有一个比较深入的理解了。在此基础之上，authorization server与resource server的具体实现方式已经不那么重要了，因为很多细节问题一般都会被相关框架屏蔽。因此，这篇文章跳过具体实现，从高一层次来了解一下authorization server与resource server的实现中需要解决什么问题。


# 数据结构

---

authorization server需要能够生成authorization grant、access token，同时需要记录client的相关信息，所以在数据库中这些记录是不可少的。参照Flask-OAuthlib的文档，以下数据结构是必不可少的。

{% highlight python %}

class User(db.Model):

    id = db.Column(db.Integer, primary_key=True)


class Client(db.Model):

    client_id = db.Column(db.String(40), primary_key=True)
    client_secret = db.Column(db.String(55), unique=True, index=True,
                              nullable=False)
    is_confidential = db.Column(db.Boolean)

    _redirect_uris = db.Column(db.Text)
    _default_scopes = db.Column(db.Text)

    @property
    def default_redirect_uri(self):
        pass


class Grant(db.Model):
    id = db.Column(db.Integer, primary_key=True)
    user_id = db.Column(db.Integer,
                        db.ForeignKey('user.id', ondelete='CASCADE'))
    client_id = db.Column(db.String(40), db.ForeignKey('client.client_id'),
                          nullable=False)
    client = db.relationship('Client')

    code = db.Column(db.String(255), index=True, nullable=False)

    redirect_uri = db.Column(db.String(255))
    expires = db.Column(db.DateTime)

    _scopes = db.Column(db.Text)


class Token(db.Model):
    id = db.Column(db.Integer, primary_key=True)
    client_id = db.Column(db.String(40), db.ForeignKey('client.client_id'),
                          nullable=False)
    user_id = db.Column(db.Integer, db.ForeignKey('user.id'))

    access_token = db.Column(db.String(255), unique=True)
    refresh_token = db.Column(db.String(255), unique=True)
    expires = db.Column(db.DateTime)
    _scopes = db.Column(db.Text)


{% endhighlight %}

对于authorization grant而言，一般都是短时间内有效、只能使用一次的，所以可以放倒cache中。对于access token而言，以上示例仅仅适用于bearer token，对于其他类型的token就要视具体情况而定了。

# endpoint

---

OAuth 2.0规范中规定authorization server相关的endpoint有两个。在不同的授权方案中，并不是所有的endpoint都会用到，所以可以有选择性地实现，不过一般情况两个都会实现。所谓的endpoint就是authorization server中提供的uri，client需要向这些uri发送请求。

1. authorization endpoint：需要授权时，resource server会把用户重定向到这个uri。
2. Token endpoint：resource server通过这个uri来获得access token。

# scope管理与resource server鉴权

---

OAuth 2.0中利用scope来确定权限范围，规定哪些资源有权访问，这在授权系统中肯定是不可或缺的。但是，对于如何对scope进行设定、scope与权限的映射关系是怎样的，规范中并没有设定。所以，scope与权限的管理是实现中必须要做的，并且实现方式根据具体情况而定。

对应的，resource server需要根据scope来进行鉴权，具体实现方式也是和scope的设计高度相关的。对于接口倒可以和传统的实现方式一样，如下所示：

{% highlight python %}

@app.route('/api/me')
@oauth.require_oauth('email')
def me():
    return "ok"

{% endhighlight %}

# 分离与合并

---

在传统的“认证即授权”模式中，认证部分和授权部分一般是一体的，例如两者在同一个进程当中，读取同一个数据库。只不过authorization server会读取user对应的表，resource server会根据user_id去查找权限相关的表。

在OAuth 2.0中，authorization server与resource server可以是一体的，也可以是分开的。如果放到一起，那和传统的实现方式没有本质区别。如果分开放（分别在不同的进程或者不同的机器），那么实现策略可以有如下两种：

1. resource server根据user_id去查找权限相关的表，当然，authorization server也会去读取一样的内容。这种方式仅仅用bearer token就可以了。
2. 具体权限信息放倒token当中，resource server不需要去读数据库查询具体权限，仅需要鉴定token就行。此时，bearer token无法满足要求，需要设计其他类型的token。

(the end)
