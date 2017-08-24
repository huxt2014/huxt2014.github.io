---
layout: post
title:  "Latex编辑数学公式样例"
date:   2017-08-24 21:00:00 +0800
categories: other
tags: other
---

---

![](/images/latex-math-formular/1.png)

{% highlight text %}
\max \limits_{b, \boldsymbol{w}} \quad &margin(b,\boldsymbol{w})\\
subject \ to \quad &y_{n}(\boldsymbol{w^{T}x}_{n}+b) > 0 \quad n=1,\dots,N\\
&margin(b, \boldsymbol{w})=\min \limits_{n=1,\dots,N} \frac{1}{||\boldsymbol{w}||}y_{n}(\boldsymbol{w^{T}x}_{n}+b)

说明：
\limits_{}  ->  max的下标
\\  ->  换行
\quad  ->  空格1m宽度
\qquad  ->  空格2m宽度
\  ->  空格0.3m宽度
\boldsymbol{}  ->  加粗若干个字母
&  ->  对齐

{% endhighlight %}

---

![](/images/latex-math-formular/2.png)

{% highlight text %}
\sum_{n=1}^{N} y_{n}\alpha_{n} = 0

说明：
\sum  -> 求和
{% endhighlight %}


---

![](/images/latex-math-formular/3.png)

{% highlight text %}

& h(k) = \lfloor m(kA\bmod 1) \rfloor \\
& h(k) = \lceil kA \rceil

{% endhighlight %}
