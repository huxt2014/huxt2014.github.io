---
layout: post
title:  "深入scikit-learn：decision tree and CART"
date:   2017-08-11 21:00:00 +0800
categories: machine-learning
tags: machine-learning
---

决策树算法是一种即可用于classification，也可用于regression的一类算法。本文基于sklearn中决策树算法的实现，用伪代码简要记录了sklearn中CART的实现方法。

# 理论回顾

---

决策树算法非常直接，相当于是一个贪心算法，相比SVM而言没有太多的数学推导与证明过程。其主要思想是，给定一个打好标签的样本集，将其划分为n个不相交子集，尽量使得每个子集中的样本有相同的标签。如果所得到子集中有不同标签的样本，那么对该子集再继续划分，直到所得到的子集中都是同标签的样本为止。记录下每次划分的集合和划分参数，那么可以得到一个树结构。在预测的时候，依照划分参数从树的根节点一直走到叶子节点，叶子节点对应子集的标签就是预测的结果。

在大多数决策树算法的实现中，每次划分都会划分为两个子集，那么最后结果会得到一颗二叉树。而划分的方法一般采用在样本集的第 ![](/images/sklearn-cart/01.png) 个属性上选取一个thredhold ![](/images/sklearn-cart/02.png) ，在 ![](/images/sklearn-cart/01.png) 属性上小于thredhold的样本划分到左子节点，反之划分到右子节点。划分的标准是计算子集的不纯度，使得不纯度最小。

记第m个节点为 ![](/images/sklearn-cart/1.png) ，该节点任意一个划分参数记为 ![](/images/sklearn-cart/2.png)，那么划分参数 ![](/images/sklearn-cart/03.png) 所划分的左右节点可以表示为：

![](/images/sklearn-cart/3.png) 

记不纯度函数为 ![](/images/sklearn-cart/4.png) （可以有多种选择），那么m节点上最终划分参数的计算方法为：

![](/images/sklearn-cart/5.png) 

# sklearn实现

---

比较有名的决策树算法有ID3、C4.5、C5.0、CART等，sklearn中实现的是经过了优化的CART，其主要部分由Cython编写。该实现中提供了两种训练策略：深度优先和最佳purity优先，以下伪代码描述的是最佳purity优先策略的实现。


{% highlight python %}

# assume that these are global variables
validate X
validate Y
init tree
init priority_heap

build()

def build():

    # Each segment correspond to a node(either leaf or not) of the tree.
    # If not a leaf, a split will be taken and then the segment will record
    # the split information.

    # the whole samples as the first segment, will be set as root
    first_segment = add_split_node(0, sample_length, left)
    priority_heap.push(first_segment)

    while not priority_heap.empty:
        segment = priority_heap.pop()
        node = tree[segment.node_id]

        if not segment.is_leaf:
            segment_left = add_split_node(node, segment.start, segment.pos,
                                          left)
            segment_right = add_split_node(node, segment.pos, segment.end,
                                           right)

            priority_heap.push(segment_left)
            priority_heap.push(segment_right)


def add_split_node(parent_node, start, end, left_or_right):

    is_leaf = check()

    if is_leaf:
        # do split and get feature id, threshold and so on
        split = node_split(start, end)
        is_leaf = check_again()

    # add node to tree
    child_node_id = add_node(parent_node, left_or_right, is_leaf, 
                             split.feature_index, split.threshold)

    seg.node_id = child_node_id
    seg.start = start
    seg.end = end
    seg.pos = split.pos
    seg.is_leaf = is_leaf

    return seg

# best first search
# sort the feature value and find the threshold for each feature
def node_split(start, end):
    
    # use a Fisher-Yates liked algorithm to select feature, but a bit more
    # complex. The feature list will store:
    #     drawn and known constant features
    #     not drawn and constant features
    #     not drawn and newly found constant features
    #     not drawn and not constant features
    #     drawn and not constant features.
    # for simplex, the following description will skip these details

    init current_split, best_split, current_purity, best_purity

    while has non-constant features not checked:
        # current feature index
        f_i <- select a feature
        current_split.feature_index = f_i

        # copy the feature value of the current feature of [start: end]
        # samples. The length of v_list is still the same with total
        # samples' length
        get v_list

        # actually there is a presort option, which will skip this step.
        # but I just omit it for simplex.
        sort v_list[start : end], x[start : end] in-place

        if v_list[end -1] < v_list[start] + very_small_value:
            constant_features.add(f_i)
        else:
            p = start
            while p < end:
                p+=1
                current_split.pos = p
                # calculate purity for each pos, consuming more time than
                # gradient boosting.
                current_purity = get_purity()
                if current_purity > best_purity:
                    current_split.threshold = (v_list[p-1] + v_list[p]) / 2
                    best_split = current_split

    # after searching the best split finished, reorganize into 
    # [start: pos] and [pos: end], just like quick sort's partition
    do_partition

    return best_split

{% endhighlight %}

以上伪代码相比源代码已经省略了茫茫多的参数校验、参数传递、边界条件判定以及若干实现细节，主要是梳理一下实现的思路。梳理完成后，时间复杂度就比较好分析了。由于是树结构，所以树的层数为 ![](/images/sklearn-cart/6.png) ；每层上计算某个feature的最佳threshold时，每个节点作为划分时都要计算一次purity，所以每层要计算 ![](/images/sklearn-cart/7.png) 次；要遍历每个feature，所以要再乘个 ![](/images/sklearn-cart/8.png) ，最后得到时间复杂度是 ![](/images/sklearn-cart/9.png) 。这个时间复杂度是没有考虑排序耗时的，如果没有提前排序，那么复杂度会再高一点。 

最佳purity优先策略中，为了找到最佳threshold，需要耗费 ![](/images/sklearn-cart/7.png) 的时间。如果不需要逐个计算呢，那么这部分就可以省了。所以，sklearn还提供了gradient boosting的方法来随机选择一个threshold，达到加速训练的目的。该方法仅仅是在node_split函数中有所不同，概括如下：

{% highlight python %}

# so called gradient boosting
# chose a random value as threshold for each feature
def node_split(start, end):

    init current_split, best_split, current_purity, best_purity

    while has non-constant features not checked:

        f_i <- select a feature
        current_split.feature_index = f_i

        min_value, max_value <- find min and max value for f_i
        current_split.threshold = random(min_value, max_value)

        do_partition
        current.pos <- get partition position

        # evaluate
        current_purity = get_purity()
        if current_purity > best_purity:
            best_split = current_split

    do_partition

    return best_split

{% endhighlight %}

gradient boosting的实现方法中，在计算某个一feature的最佳threshold时，没有遍历每个划分方式，仅仅是在feature value的最大值和最小值之间随机选择一个值作为threshold，所以时间复杂度可以降低为 ![](/images/sklearn-cart/10.png)。

在阅读源码的时候，还有一些实现的细节是值得一提的，概括如下：
1. sklearn的实现中主要有三个类：tree_builder，splitter，criterion。tree_builder主要负责深度优先／最佳purity优先的控制，splitter主要负责划每个节点的划分参数选择，criterion提供了多种impurity函数的实现。
2. 树结构是用数组实现的，而不是用链表。tree[0]为树的root节点，每个节点记录了左右子节点在数组中的index。由于这个数组可以视为一串存储了相同数据类型的连续内存，所以sklearn在内部可以将其包装成numpy的ndarray。
3. 在递归划分的时候，sklearn没有采用递归函数来实现。在最佳purity优先策略中用到了priority heap，在深度优先策略中用到了stack。
4. 由于sklearn的实现中深度使用了ndarray，所以对于二维数组的操作都是在一维数组上进行的。类似于numpy的stride，sklearn使用了各种stride：sample_stride、y_stride、feature_stride、value_stride等等。
5. 在排序的时候，没有直接对原始的样本进行排序，而是建立索引后对索引排序。例如，原始数据为[4,5,2,3,1]，建立索引[1,2,3,4,5]，由小到大排序后的索引是[5,3,4,1,2]，原始数据没有变化。
6. sklearn的实现没有提供修剪操作，但是在训练的时候可以调整最大深度、最大叶子节点数、最小分割样本数、最小impurity等，通过这样的方式同样可以达到regularization的目的。





