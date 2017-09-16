---
layout: post
title:  "OAuth2.0：client实现"
date:   2017-09-15 21:00:00 +0800
categories: auth
tags: auth
---

[上一篇文章]({{site.baseurl}}{% post_url 2017-09-14-oauth2-protocol %})介绍了OAuth 2.0的基本交互流程，但是仅仅看RFC差那么一点意思，所以这篇文章通过一个client实现示例来进一步探索一下OAuth 2.0是如何实现的。在这篇文章的示例中，一个Web应用（client）需要访问我（resource owner）在github（resource server）上的个人信息，因此需要向github授权中心（authorization server）申请获得access token。

# 注册client

---

如规范所说，client在发起OAuth授权之前，应该向authorization server注册自己。如果你有github账号，那么可以在自己账户的settings中找到developer settings，在其中OAuth Apps选项中就可以完成这个注册了。如下图所示，关键的信息就是Application name和callback URL了。

![](/images/oauth2-client/github-register.png)

在注册成功后，可以获得client id和client secret，用于client与authorization server之间的认证。

# client实现

---

web应用由Flask实现，借助Flask-OAuthlib来提供OAuth 2.0的支持。client提供了/github_info接口，访问这个接口的时候如果还没有拿到token，那么会重定向到Github的认证界面，认证完成之后会重新重定向回来。

{% highlight python %}

from flask import Flask, session, request, jsonify, url_for, redirect
from flask_oauthlib.client import OAuth

app = Flask(__name__)
app.secret_key = 'development'
oauth = OAuth(app)
github = oauth.remote_app(
    'github',
    consumer_key='ce313306051273a4005a',
    consumer_secret='abdeb4714924cc87c147c7a2a2d6018b0e5840bf',
    request_token_params={'scope': 'user:email'},
    base_url='https://api.github.com/',
    request_token_url=None,
    access_token_method='POST',
    access_token_url='https://github.com/login/oauth/access_token',
    authorize_url='https://github.com/login/oauth/authorize'
)


@app.route('/github_info')
def github_info():
    if 'github_token' in session:
        me = github.get('user')
        return jsonify(me.data)

    return redirect(url_for('github_auth'))


@app.route("/github_auth")
def github_auth():
    return github.authorize(callback=url_for('authorized', _external=True))


@app.route("/authorized")
def authorized():
    resp = github.authorized_response()
    if resp is None or resp.get('access_token') is None:
        return 'Access denied: reason=%s error=%s resp=%s' % (
            request.args['error'],
            request.args['error_description'],
            resp
        )
    session['github_token'] = (resp['access_token'], '')
    return redirect(url_for("github_info"))


@github.tokengetter
def get_github_oauth_token():
    return session.get('github_token')


app.run(host="0.0.0.0", port=5000)

{% endhighlight %}

启动之后来看看具体效果。第一次访问/github_info时会被重定向到如下界面：

![](/images/oauth2-client/github-auth.png)

授权完成后会回到/github_info，并且能够拿到github的个人信息。在github上可以看到我的账户给了这个client授权，并且可以取消掉，如下图所示。

![](/images/oauth2-client/github-granted-app.png)

# 交互过程解剖

---

在web应用的后台可以看到各种302的日志，为了看到更详细的信息，可以利用wireshark来抓包看看，关键部分如下所示。

{% highlight text %}

GET /github_info HTTP/1.1
Host: 192.168.1.2:5000

HTTP/1.0 302 FOUND
Location: http://192.168.1.2:5000/github_auth

GET /github_auth HTTP/1.1
Host: 192.168.1.2:5000

HTTP/1.0 302 FOUND
Location: https://github.com/login/oauth/authorize?response_type=code
&client_id=ce313306051273a4005a
&redirect_uri=http%3A%2F%2F192.168.1.2%3A5000%2Fauthorized&scope=user%3Aemail

missing 1...

GET /authorized?code=686e1bcc5aa8905aad3a HTTP/1.1
Host: 192.168.1.2:5000

HTTP/1.0 302 FOUND
Location: http://192.168.1.2:5000/github_inf

missing 2...

GET /github_info HTTP/1.1
Host: 192.168.1.2:5000

missing 3...

HTTP/1.0 200 OK
Content-Length: 1271

{% endhighlight %}

结合OAuth 2.0规范和web应用的源码，以上交互过程很容易理解，只是相对于完整的交互过程，上面的过程少了三个部分。第一个部分是重定向到authorization server进行认证时，有一个认证过程，通过之后才会重定向到/authorized。第二个部分是拿到code之后换取token；第三个部分是通过token获取信息。

之所以会缺失，是因为这三个过程都经过了加密，无法看到具体过程。不过，对于第二、第三部分可以通过分析Flask-OAuthlib的源码得到。这两个部分均由web应用主动发起，具体请求如下所示：

{% highlight text %}

#用code交换token
url:     https://github.com/login/oauth/access_token
method:  POST
headers: {'Content-Type': 'application/x-www-form-urlencoded'}
body:    'grant_type=authorization_code&client_secret=abdeb4714924cc87c147c7a2a
          2d6018b0e5840bf&client_id=ce313306051273a4005a
          &redirect_uri=http%3A%2F%2F172.30.21.70%3A5000%2Fauthorized
          &code=17f3193a159d6b1bcf3e'

# 用token获取信息
url:     https://api.github.com/user
method:  GET
headers: {'Authorization': 'Bearer ae4f774e317ef1acc0881a2b93fb3cd0177cdaf7'}

{% endhighlight %}

# 小结

---

从完整流程可以看到，在这个示例中，web应用使用了Authorization Code方式完成授权，token放在了headers中，申请的scope为"user:email"。

在实际应用当中，作为Web应用，获得token后应该存储在server端，而不是放在session中。

(the end)
