---
layout: post
title:  "深入scikit-learn：基于svm的分类"
date:   2017-08-01 21:00:00 +0800
categories: machine-learning
---

scikit-learn封装了libsvm和liblinear以提供SVM的使用接口，可以用于分类和回归。libsvm和liblinear均由台湾国立大学的Machine Learning and Data Mining Group主导开发。本文基于台湾国立大学林轩田教授在coursera.org上开设的《机器学习技法》课程，简要回顾了使用SVM进行分类的理论基础，并且探究了一下scikit-learn是如何封装libsvm的。

Stephen Boyd的《Convex Optimization》中也有对SVM的介绍，但是用来描述最优化问题的公式与《机器学习技法》课程中略有不同，并且没有对核函数进行深入介绍，也没有代码实现。相比而言，《机器学习技法》课程的理论介绍与libsvm的代码实现匹配度更高。

# 理论回顾

---

svm分类的基本思想是将已经打好标签的A、B两类样本当作空间中的点／向量，然后找到一个合适的超平面将这些点分割到两个子空间当中。如果这些点可以分割（A类全部在一个子空间当中，B类全部在另一个子空间当中），那么找到这样一个平面，使得离平面最近的那个点到平面的距离尽可能地远（hard margin）；如果不可分割（至少存在一类点被分割到两个子空间中），那么找到这样一个平面，使得把点分到错误子空间中的错误程度尽可能地小（soft margin）。

#### hard margin

记超平面为 ![](/images/sklearn-svm-classification/1.png) ，任意一个样本表示为 ![](/images/sklearn-svm-classification/2.png) ， ![](/images/sklearn-svm-classification/01.png) 有d个维度， ![](/images/sklearn-svm-classification/02.png) 为标签（1或-1），共有N个样本，如果可以分割，那么该问题可以描述为：

![](/images/sklearn-svm-classification/3.png) 

经过推导转化后，以上原始问题可以转化为如下二次规划问题。这个二次规划问题中，一共有d+1个变量和N个约束条件。

 ![](/images/sklearn-svm-classification/4.png) 

#### 映射到Z空间的hard margin

很多情况下样本在原始空间中不可分割，但是映射到更高维空间中可能可以分割，例如将原始空间的向量 ![](/images/sklearn-svm-classification/5.png) 映射到Z空间当中的向量 ![](/images/sklearn-svm-classification/6.png) 。在Z空间中对问题进行求解的过程本质上是和上述过程一样的，仅需将上述问题中的 ![](/images/sklearn-svm-classification/01.png) 替换成 ![](/images/sklearn-svm-classification/03.png) 就可以了。但是，Z空间的维度更高，那么上述二次规划问题中的变量更多（ ![](/images/sklearn-svm-classification/04.png) 的维度增加了），随之带来的是计算量增加的问题，而利用拉格朗日对偶以及KKT条件可以部分解决该问题。Z空间中上述二次规划问题的对偶问题如下，也是一个二次规划问题：

![](/images/sklearn-svm-classification/7.png)

其中，目标函数还有如下两种表达方式：
1. 展开形式：![](/images/sklearn-svm-classification/8.png)
2. 矩阵形式：![](/images/sklearn-svm-classification/9.png)


新的二次规划问题有N个变量，N+1个约束条件，这样二次规划问题的复杂度就和Z空间的维度无关了。但是，目标函数中存在 ![](/images/sklearn-svm-classification/05.png) 这样的计算项，所以计算量与Z空间的维度还是没有完全解耦，而核函数可以解决这个问题。核函数的基本思想是 ![](/images/sklearn-svm-classification/10.png) ， ![](/images/sklearn-svm-classification/06.png) 计算量小很多且与Z空间维度无关。

以二次多项式核函数为例，原理如下。d维度的向量映射到Z空间后的维度是 ![](/images/sklearn-svm-classification/07.png) 的，所以计算  ![](/images/sklearn-svm-classification/05.png) 的复杂度也是 ![](/images/sklearn-svm-classification/07.png) 的。但是利用核函数后，计算复杂度是  ![](/images/sklearn-svm-classification/08.png)。 

![](/images/sklearn-svm-classification/11.png)

![](/images/sklearn-svm-classification/12.png)

除了求解二次规划问题的过程外，利用 ![](/images/sklearn-svm-classification/04.png) 、b进行预测的时候，所有在Z空间中的计算都可以用 ![](/images/sklearn-svm-classification/06.png) 来表示。这样，所有的计算就只需要考虑 ![](/images/sklearn-svm-classification/06.png) 的复杂度，不需要再进行 ![](/images/sklearn-svm-classification/05.png) 的计算了。

#### Z空间中的soft margin

如果即使在Z空间中依然不可分割，那么该怎么办呢，那么就放宽约束条件以及在目标函数添加penalty吧。修改后的原始问题和对偶形式如下所示，对偶形式中有N个变量和2N+1个约束。。
* 原始问题：

![](/images/sklearn-svm-classification/13.png) 
* 对偶形式：

![](/images/sklearn-svm-classification/14.png) 

可以看到，soft-margin与hard-margin没有本质的不同，核函数的概念也是通用的。使用SVM时，一般情况下都是用的soft-margin SVM，libsvm中也是如此。

# sklearn 实现

---

利用拉格朗日对偶，soft-margin SVM问题被转化成了一个二次规划问题。二次规划问题属于凸优化问题，在时间复杂度方面已经处于可以接受的范围，但是如果仅仅使用通用的二次规划求解软件，那么在空间复杂度方面会碰到问题。

如果使用通用的求解软件，那么需要将矩阵Q完全计算出来。Q往往是密集矩阵，假设矩阵元素的类型是float，占用4byte，那么如果有1K个样本就需要4M内存，10K个样本需要400M内存，100K个样本需要40G内存。为了达到比较好的训练效果，实践中往往会使用核函数，这相当于增加了Z空间的维度，即增加了VC维度，那么往往需要大数量的样本来避免过拟合问题。所以，实际使用当中Q矩阵往往占用空间过大，难以使用通用的二次规划软件来求解问题。

为了避免完全存储Q，可以使用Sequential Minimal Optimization (SMO)算法来求解二次规划问题。该方法的思路是每次迭代过程中仅仅考虑两个变量： ![](/images/sklearn-svm-classification/15.png) ，那么每次迭代过程中仅仅需要矩阵Q的一部分。libsvm中使用的就是这一算法。

在libsvm的官网可以看到怎样使用Python调用libsvm的样例，即把libsvm编译成动态链接库后，通过ctypes模块加载。sklearn中使用Cython将libsvm包装了一下，使得调用接口更加友善，其具体做法是将数据保存到numpy.ndarray中，然后将numpy.ndarray中数据块的地址传到libsvm中。此外，sklearn中还扩充了libsvm，使其可以用于稀疏矩阵。

libsvm中求解上述二次规划问题的实现中主要包含两个部分：SMO和cache。对于SMO部分的实现，如果没有数学公式与伪代码，直接看C++源码简直是看天书，所以等有机会做些铺垫再补充吧，在这里主要谈谈cache。

如果有l个样本，那么cache中存储了长度为l的数组head，head[i]存储了Q矩阵的第i列。考虑到需要动态计算，所以数组中的元素是一个结构体，记录了数据块的指针和长度。同时，head还能组织成循环列表，每次读取第i列时，head[i]会被移到链表的头部，如果cache的空间不够了，那么会释放链包尾部的那一列。所以，cache的结构如下所示：

{% highlight C %}
class Cache
{
...
private:
    int l;                      //行列数
    long int size;              //最大存储的float的个数
    struct head_t
    {
        head_t *prev, *next;	// for circular list
        float *data;
        int len;
    };
    head_t *head;               //数组
    head_t lru_head;            //lru_head.next指向即将被释放的列
...
};

{% endhighlight %}

可以认为，当cache的size最小时（缓存两列），那么在迭代过程中Q的值全部通过计算得到；如果cache的size足够大能够存下Q，那么Q只需要计算一次，迭代过程中只需读取内存就行了。这也就意味着cache是用时间来换空间。所以，在实际使用中，如果发现训练速度过慢，可以尝试调大cache来尝试加快速度。

此外，考虑到核函数、VC维度、样本数之间的关系，使用简单的核函数或者不使用核函数可以减少所需要的样本数，以减少内存压力。