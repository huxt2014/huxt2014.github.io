---
layout: post
title:  "深入scikit-learn：SGD for linear regression"
date:   2017-12-04 21:00:00 +0800
categories: machine-learning
tags: machine-learning
---

这篇文章主要来看一下sklearn中用于linear regression的stochastic gradient descent是如何实现的，对应的代码位置为sklearn.linear_model.stochastic_gradient.SGDRegressor。

# 理论回顾

---

在linear regression问题中，我们需要解决如下问题：

![](/images/sklearn-sgd-linear-regression/01.png)

上述目标函数称为cost function，其中![L] 称为loss function，![alpha]与![R]用于regularization。作为线性模型，本来还应该有一个b，这里先省略，下文再细说。

在gradient descent中，每次对![w]更新的量为上述目标函数的梯度。稍微观察一下可以发现，带![sum] 的那一项其实就是在每个样本的loss function上计算梯度后取平均值，所以有如下结果:

![](/images/sklearn-sgd-linear-regression/02.png)

<p style="display:none">
\boldsymbol{w}_{t+1} & =  \boldsymbol{w}_{t} - \eta(\frac{1}{n} \frac{\partial \sum_{i=1}^{n}L(\boldsymbol{w}_t^T\boldsymbol{x}_{i}, \ y_{i})}{\partial \boldsymbol{w}_t} + \alpha \frac{\partial R(\boldsymbol{w}_t)}{\partial \boldsymbol{w}_t}  ) \\
& =  \boldsymbol{w}_{t} - \eta(\frac{1}{n} \sum_{i=1}^{n}\frac{\partial L(\boldsymbol{w}_t^T\boldsymbol{x}_{i}, \ y_{i})}{\partial \boldsymbol{w}_t} + \alpha \frac{\partial R(\boldsymbol{w}_t)}{\partial \boldsymbol{w}_t}  )
</p>

由上述等式可以看出，在每次迭代过程中，我们需要计算所有样本loss function的梯度，然后求平均值。在数据量一般的时候还好，在数据量特别大的时候，每次更新![w] 都需要遍历所有样本有点无法接受，所以有了stochastic gradient descent。

在stochastic gradient descent中，我们每次对![w] 更新的量不是所有样本loss function梯度的平均值，而是某个样本loss function的梯度。同时，考虑到线性模型的特殊性，loss function的梯度是可以做一下简化的。最终结果如下所示：

![](/images/sklearn-sgd-linear-regression/03.png)

<p style="display:none">
\boldsymbol{w}_{t+1} & =  \boldsymbol{w}_{t} - \eta(\frac{\partial L(\boldsymbol{w}_t^T\boldsymbol{x}_{i}, \ y_{i})}{\partial \boldsymbol{w}_t} + \alpha \frac{\partial R(\boldsymbol{w}_t)}{\partial \boldsymbol{w}}_t  ) \\
& = \boldsymbol{w}_{t} - \eta(\frac{\mathrm{d} L_{(\boldsymbol{w}_t^T\boldsymbol{x}_{i}, \ y_{i})}}{\mathrm{d} (\boldsymbol{w}_t^T\boldsymbol{x}_{i})}\boldsymbol{x}_{i} + \alpha \frac{\partial R(\boldsymbol{w}_t)}{\partial \boldsymbol{w}_t}  )
</p>

在实际使用当中，能够调整的参数就是learning rate ![eta] 、regularization相关的![R]与![alpha]以及loss function ![L]。下面来看看sklearn的实现。

# Regularization

---

#### 1. L1 regularization

对于L1 regularization， ![w]的迭代过程可以表述如下：

![](/images/sklearn-sgd-linear-regression/07.png)

<p style="display:none">
\boldsymbol{w}_{t+1}  & =  \boldsymbol{w}_{t} - \eta(\frac{\mathrm{d} L_{(\boldsymbol{w}_t^T\boldsymbol{x}_{i}, \ y_{i})}}{\mathrm{d} (\boldsymbol{w}_t^T\boldsymbol{x}_{i})}\boldsymbol{x}_{i} + \alpha \frac{\partial | \boldsymbol{w}_t|}{\partial \boldsymbol{w}_t}  ) \\
&= \boldsymbol{w}_{t} - \eta \frac{\mathrm{d} L_{(\boldsymbol{w}_t^T\boldsymbol{x}_{i}, \ y_{i})}}{\mathrm{d} (\boldsymbol{w}_t^T\boldsymbol{x}_{i})}\boldsymbol{x}_{i} - \eta \alpha  \mathrm{sign}(\boldsymbol{w}_t)
</p>

在stochastic gradient descent中，每次迭代都是随机选取一个点（或者少量点），所以在每次迭代中![w] 的每个分量并不一定是向最优解靠拢的。也就是说，![w] 的每个分量可能会有一个波动的过程，或者可以称为noisy。在特征数量很多的时候，直接使用上式进行迭代会存在两个问题。第一个问题是在每次更行![w] 时，几乎要更新![w] 的所有分量，这会降低迭代的速度。第二个问题是，最后得到的![w] 可能不够简洁，也就是说非零分量太多。这对这两个问题，有了对述迭代过程的改进版：

![](/images/sklearn-sgd-linear-regression/08.png)

<p style="display:none">
&\boldsymbol{w}_{t+\frac{1}{2}}   =   \boldsymbol{w}_{t} - \eta \frac{\mathrm{d} L_{(\boldsymbol{w}_t^T\boldsymbol{x}_{i}, \ y_{i})}}{\mathrm{d} (\boldsymbol{w}_t^T\boldsymbol{x}_{i})}\boldsymbol{x}_{i} \\
& for \ each \ feature: \
w_{t+1} = \begin{cases}
\mathrm{max}(0, w_{t+\frac{1}{2}} - \eta \alpha) & for \ w_{t+\frac{1}{2}} > 0 \\
\mathrm{min}(0, w_{t+\frac{1}{2}} + \eta \alpha) & for \ w_{t+\frac{1}{2}} < 0
\end{cases}
</p>

上述改进版对![w]有较多非零分量这一问题解决地不是很好，因此又有了如下改进版，也就是sklearn中采用的方法：

![](/images/sklearn-sgd-linear-regression/09.png)

<p style="display:none">
&\boldsymbol{w}_{t+\frac{1}{2}}   =   \boldsymbol{w}_{t} - \eta \frac{\mathrm{d} L_{(\boldsymbol{w}_t^T\boldsymbol{x}_{i}, \ y_{i})}}{\mathrm{d} (\boldsymbol{w}_t^T\boldsymbol{x}_{i})}\boldsymbol{x}_{i} \\
& for \ each \ feature: \
w_{t+1} = \begin{cases}
\mathrm{max}(0, w_{t+\frac{1}{2}} - (u_k + q_{k-1})) & for \ w_{t+\frac{1}{2}} > 0 \\
\mathrm{min}(0, w_{t+\frac{1}{2}} + (u_k - q_{k-1})) & for \ w_{t+\frac{1}{2}} < 0
\end{cases}
</p>

其中，![](/images/sklearn-sgd-linear-regression/10.png) ，![](/images/sklearn-sgd-linear-regression/11.png) 。


#### 2. L2 regularization

对于L2 regularization，![w]的迭代过程可以表示如下：

![](/images/sklearn-sgd-linear-regression/04.png)

<p style="display:none">
\boldsymbol{w}_{t+1}  & = (1-\eta \alpha)\boldsymbol{w}_{t} - \eta \frac{\mathrm{d} L_{(\boldsymbol{w}_t^T\boldsymbol{x}_{i}, \ y_{i})}}{\mathrm{d} (\boldsymbol{w}_t^T\boldsymbol{x}_{i})}\boldsymbol{x}_{i} 
</p>

这是一个相对简单的迭代过程，但是sklearn在实现![w] 的时候做了一些优化。在sklearn的实现当中，![w] 被表示成![w'] 与一个衰减系数wscale，每次迭代的时候也是更新这两个量，如下所示：

![](/images/sklearn-sgd-linear-regression/06.png)

<p style="display:none">
& \boldsymbol{w}_{t+1}^{'} =\boldsymbol{w}_{t}^{'} - \eta \frac{\mathrm{d} L_{((1-\eta \alpha)^{t}\boldsymbol{w}_{t}^{'T}\boldsymbol{x}_{t}, \ y_{t})}}{(1-\eta \alpha)^{t+1}\mathrm{d} ((1-\eta \alpha)^{t}\boldsymbol{w}_{t}^{'T}\boldsymbol{x}_{t})}\boldsymbol{x}_{t} \\
& wscale_{t+1} = (1-\eta \alpha)^{t+1}
</p>


粗略判断，优化后每次迭代时少计算一次向量的数乘。上述优化的推导过程如下：

![](/images/sklearn-sgd-linear-regression/05.png)

<p style="display:none">
& \boldsymbol{w}_{1}  = (1-\eta \alpha)\boldsymbol{w}_{0} - \eta \frac{\mathrm{d} L_{(\boldsymbol{w}_0^T\boldsymbol{x}_{0}, \ y_{0})}}{\mathrm{d} (\boldsymbol{w}_0^T\boldsymbol{x}_{0})}\boldsymbol{x}_{0} = (1-\eta \alpha)(\boldsymbol{w}_{0} - \eta \frac{\mathrm{d} L_{(\boldsymbol{w}_0^T\boldsymbol{x}_{0}, \ y_{0})}}{(1-\eta \alpha)\mathrm{d} (\boldsymbol{w}_0^T\boldsymbol{x}_{0})}\boldsymbol{x}_{0}) = (1-\eta \alpha)\boldsymbol{w}_{1}^{'} \\
& \boldsymbol{w}_{2} = (1-\eta \alpha)\boldsymbol{w}_{1} - \eta \frac{\mathrm{d} L_{(\boldsymbol{w}_1^T\boldsymbol{x}_{1}, \ y_{1})}}{\mathrm{d} (\boldsymbol{w}_1^T\boldsymbol{x}_{1})}\boldsymbol{x}_{1} = (1-\eta \alpha)^{2}(\boldsymbol{w}_{1}^{'} - \eta \frac{\mathrm{d} L_{((1-\eta \alpha)\boldsymbol{w}_{1}^{'T}\boldsymbol{x}_{1}, \ y_{1})}}{(1-\eta \alpha)^{2}\mathrm{d} ((1-\eta \alpha)\boldsymbol{w}_{1}^{'T}\boldsymbol{x}_{1})}\boldsymbol{x}_{1}) = (1-\eta \alpha)^2\boldsymbol{w}_{2}^{'} \\
&... \\
& \boldsymbol{w}_{t+1} = (1-\eta \alpha)\boldsymbol{w}_{t} - \eta \frac{\mathrm{d} L_{(\boldsymbol{w}_t^T\boldsymbol{x}_{t}, \ y_{t})}}{\mathrm{d} (\boldsymbol{w}_t^T\boldsymbol{x}_{t})}\boldsymbol{x}_{t} = (1-\eta \alpha)^{t+1}(\boldsymbol{w}_{t}^{'} - \eta \frac{\mathrm{d} L_{((1-\eta \alpha)^{t}\boldsymbol{w}_{t}^{'T}\boldsymbol{x}_{t}, \ y_{t})}}{(1-\eta \alpha)^{t+1}\mathrm{d} ((1-\eta \alpha)^{t}\boldsymbol{w}_{t}^{'T}\boldsymbol{x}_{t})}\boldsymbol{x}_{t}) \\
& \quad \quad = (1-\eta \alpha)^{t+1}\boldsymbol{w}_{t+1}^{'}
</p>


# learning rate的选择

---

#### 1. constant

需要传入![eta] 的初始值，在迭代的过程中![eta] 的值保持不变。

#### 2. inverse scaling

需要传入![eta] 的初始值，以及一个指数，按照如下方法计算：

![](/images/sklearn-sgd-linear-regression/inverse_scaling.png)


#### 3. optimal

需要更具loss function和 ![alpha] 来计算一个![eta] 的初始值，然后按照如下方法计算：

![](/images/sklearn-sgd-linear-regression/optimal.png)

#### 4. other

在sklearn中还有PA1与PA2两种方式，但是没有找到相关说明和文献。

# loss function的选择

---

#### 1. squared loss

![](/images/sklearn-sgd-linear-regression/squared_loss.png)

#### 2. huber loss


![](/images/sklearn-sgd-linear-regression/huber_loss.png)

<p style="display:none">
L_{\delta}(f(x), y) =\begin{cases}
\frac{1}{2}(y-f(x))^2& for \ |y-f(x)| \le \delta\\
\delta|y-f(x)|-\frac{1}{2}\delta^2 & otherwise
\end{cases}
</p>

#### 3. epsilon insensitive loss

![](/images/sklearn-sgd-linear-regression/epsilon_insensitive.png)

<p style="display:none">
L_{\epsilon}(f(x), y) =\begin{cases}
|y-f(x)| - \epsilon & for \ |y-f(x)| \ge \epsilon \\
0 & otherwise
\end{cases}
</p>

#### 4. squared epsilon insensitive loss

![](/images/sklearn-sgd-linear-regression/squared_epsilon_insensitive.png)

# Average Stochastic Gradient Descent

---

相关研究表示，在stochastic gradient descent方法中，最后的结果取迭代过程中所有![w] 的平均值也是一个不错的选择（前提是有足够多的数据量），如下所示：

![](/images/sklearn-sgd-linear-regression/12.png)

<p style="display:none">
\overline{\boldsymbol{w}}_t = \frac{1}{t-t_0}\sum_{i=t_0+1}^{t} \boldsymbol{w}_{i}
</p>

上式可以转换成一个迭代的形式，如下所示：

![](/images/sklearn-sgd-linear-regression/13.png)

<p style="display:none">
\overline{\boldsymbol{w}}_{t+1} =   \overline{\boldsymbol{w}}_t + \frac{1}{\mathrm{max}(1, t-t_0)}(\boldsymbol{w}_{t+1} - \overline{\boldsymbol{w}}_{t}) 
</p>

sklearn使用了一个优化的版本，具体参考[这篇文章](https://www.microsoft.com/en-us/research/wp-content/uploads/2012/01/tricks-2012.pdf)。

[eta]: /images/common/eta.png
[w]: /images/common/w.png
[w']: /images/common/w'.png
[sum]: /images/common/sum.png
[L]: /images/common/L.png
[R]: /images/common/R.png
[alpha]: /images/common/alpha.png
