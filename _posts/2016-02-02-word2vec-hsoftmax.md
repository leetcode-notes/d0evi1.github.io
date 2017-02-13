---
layout: post
title: word2vec中的Hierarchical Softmax
description: 
modified: 2016-02-02
tags: [word2vec+Huffman]
---

本文主要译自paper 1.

# Abstract

神经网络概率语言模型(NPLM)与过去广泛使用的n-gram语言模型相互竞争，甚至偶尔优于后者。NPLM最大的缺点是训练和测试时间超长。Morin和Bengio提出了一种层次化语言模型，它会构建一棵关于词的二叉树，它比非层次化的使用专家知识的模型快2个数量级。我们引入了一种快速的层次化语言模型，它使用一种简单的基于特征的算法，可以从数据中自动构建word树。我们接着展示，产生的模型胜过非层次化神经网络模型、n-gram模型。

# 介绍

统计语言模型的关注点是，构建基于词序列的概率模型。这样的模型可以被用于区分可能的序列与不可能的序列，广泛用于语音识别，信息检索，机器翻译。大多数统计语言模型都基于Markov猜想：一个词的分布只依赖于
在它之前出现的固定数目的词。该猜想明显是错误的，但它很简洁方便，因为它将固定长度的词序列的概率分布建模问题，缩减成：给定前面固定数目的词（称为上下文：context），建模下一个词的分布。此处，我们使用：P(wn|w1:n-1)来表示分布，wn是下一个词, w1:n-1表示上下文(w1,..,wn-1)。

目前n-gram模型是最流行的统计语言模型，因为简单，并且性能很好。这些模型的条件概率表<img src="http://www.forkosh.com/mathtex.cgi?P(w_n| w_{1:n-1})">，通过统计训练数据中的n元组，并进行归一化进行估计。因为n元组的数据是n的指数级，对raw counts进行平滑会达到很好的性能。n-gram模型有许多平滑方法，详见paper 2. 尽管n-gram模型有许多复杂的平滑方法，n-gram模型很难利用大量上下文，因为数据的稀疏性问题很严重。这种现像的主要原因是，经典的n-gram模型是本质上有界的条件概率表，里面的条目都是相互独立的。这些模型不会利用这样的事实：即相似的词常出现在相似的上下文中，因为它们没有相似性的概率。基于分类的n-gram模型（paper 3）主要解决该问题，通过将词和上下文，基于使用模式聚类到分类中，使用这些分类信息来提升泛化。它会提升n-gram的性能，这种方法引入了严格死板的相似性，因为每个词很典型，都属于确定的某个类。

另一种可选的、更灵活的抵消数据稀疏性问题的方法是，将每个词使用一个real-valued的特征向量，它会捕获该特性，以便相似的上下文中的词，具有相似的特征向量。接着，下一个词的条件概率可以被建模成一个关于上下文词和下一个词的平滑函数。这种方法提供了自动平滑，因为对于一个给定的上下文，相似的词可以保证分配到相似的概率。同样的，相似的上下文现在也可能具有相似的表示，并会生成对下一个词相似的预测。大多数基于该方法的模型，使用一个前馈神经网络(feed-forwark NN)，将上下文词汇的特征向量映射到下一个词的分布上。这种方法的可能最好模型是神经网络语言模型NPLM(paper 4)，在100w词级别的语料上，它胜过n-gram模型。

# 层次化神经网络语言模型

NPLM的主要缺点是，这样的相类似模型，训练和测试时间很慢。因为下一个词的概率计算，需要显式对词汇表中所有词进行归一化，对于给定下一个词的概率计算开销，以及对于在下一个词之上的所有分布的计算开销，事实上两者几乎一样：它们会花费关于词汇表size的线性时间开销。由于在这样的模型上，计算精确的梯度，需要重复计算给定上下文的下一个词的概率，通过增加概率来更新模型参数，训练时间与词汇表size成线性关系。通常，自然语言数据集包含着上万的词汇，这意味着即使以最简单方式训练这种类NPLM模型，在实际中的计算开销仍过大。一种加速该过程的方法是，使用专门的重要性sampling过程，来逼近要学习所需的梯度(paper 5)。然而，该方法可以本质上加速训练时间，测试时间的开销依然很快。

我们引入了层次化NPLM(paper 6)，对比普通的NPLM，它在训练和测试的时间复杂度均做到了指数级的衰减。通过二叉树替代NPLM中非结构化的词汇表，可以表示成一个层次化的词汇表词簇。每个词对应于树上的叶子节点，可以由顶点到叶子节点的路径唯一确定。如果N是词汇表的词数，并且树是一棵平衡树，任何词都可以由一串O(log N)二分决策来指定，它会指定两个子节点，当前节点可以访问到下一节点。该过程将N-way选择替换成一串O(log N)的二分选择。在概率术语中，N-way归一化，可以替换成一串O(log N)的局部(二分)归一化。结果上，词汇表中的词分布，可以通过提供访问左子节点的概率来指定。在层次化NPLM中，这样NPLM方式的局部概率的计算：采用一个特征向量作为上下文词汇，同时给当前节点的一个特征向量作为输入。下一个词的概率，由该词所对应路径的二分决策的概率指定。

当数据集包含上百万的词时，该模型的表现优于基于分类的3-gram，但比paper 6中的NPLM表现差。这种层次化NPLM模型，比普通的NPLM快2个数量级。这种方法的主要限制，主要是用于构建word树的过程。该树可以从WordNet IS-A分类体系开始，通过结合人工和数据驱动处理，并将它转换成一个二叉树。我们的目标是，将该过程替换成从训练数据中自动构建树，不需要任何专家知识。我们也探索了使用树（里面的词汇至少出现一次）的性能优点。

# log-bilinear model

我们使用log-bilinear语言模型（LBL：Paper 7）作为我们层次化模型的基础，因为它的性能十分出色，并且很简洁。类似于所有神经网络语言模型，LBL模型将每个词表示成一个real-valued的特征向量。我们将词w的特征向量表示成:<img src="http://www.forkosh.com/mathtex.cgi?r_w">，所有词的特征向量组成矩阵R. 模型的目标：给定上下文w1:n-1，预测下一个词wn。我们将要预测下一个词的的特征向量表示成<img src="http://www.forkosh.com/mathtex.cgi?\hat{r}">，它可以通过对上下文词汇的特征向量进行线性组合得到：

公式一：<img src="http://www.forkosh.com/mathtex.cgi?\hat{r}=\sum_{i=1}^{n-1}C_{i}r_{w_i}">

其中，Ci是权重矩阵，它与位置i的上下文有关。接着，要预测的特征向量，和词汇表中每个词的特征向量，两者间的相似度通过内积计算。该相似度接着进行指数化，归一化，从而获取下一个词的分布：

公式二：<img src="http://www.forkosh.com/mathtex.cgi?P(w_n=w|w_{1:n-1})=\frac{exp(\hat{r}^Tr_{w}+b_{w})}{\sum_{j}exp(\hat{r}^Tr_{j}+b_{j})}">

这里的bw是词w的偏置bias，它被用于捕获上下文独立的词频。

注意，LBL模型可以被解释成一种特殊的前馈神经网络，它只有一个线性隐层，以及一个softmax输出层。该网络的输入是上下文词汇的特征向量，而从隐层到输出层的权重矩阵，则可以简单通过特征向量矩阵R指定。该向量隐单元的激活，对应于下一个词的预测特征向量。不同于NPLM，LBL模型需要计算隐层激活，每次预测只计算一次，隐层不存在非线性。虽然LBL模型很简单，但表现很好，在相当大的数据集上，优于NPLM和n-gram模型。

# 4.层次化log-bilinear模型

我们的层次化语言模型基于该模型. 该模型的特点是，使用log-bilinear语言模型来计算每个节点的概率，并处理树中每个词多次出现率。注意，使用多个词出现率的思想，由papar 6提出，但它并没有实现。

HLBL的第一个组件是一棵关于词作为叶子的二叉树。接着，我们假设词汇表中每个词对应一个叶子结点。接着，每个词可以通过从树顶点到叶子节点的路径唯一指定。该路径本身可以编码成一串二进制字符串d(每个节点会做二分决策)，<img src="http://www.forkosh.com/mathtex.cgi?d_i=1">对应于：访问当前节点的左子节点。例如，字符串"10"表示的是：从顶点，到左子节点，然后到右子节点。这样，每个词可以被表示成一串二进制码。

HLBL模型的第二个组件是，每个节点会做出二分决策的概率模型，在我们的case中，使用了一个LBL模型的改进版本。在HLBL模型中，和非层次化的LBL类似，上下文词汇表示成real-valued特征向量。树上的每个非叶子节点，也具有一个特征向量与它相关联，它用于区分词是在该节点的左子树还是在右子树。不同于上下文词汇，被预测的词，被表示成使用它们的二进制码。然而，这种表示相当灵活，因为编码中的每种二进制数字，相当于该节点做出的决策，它依赖于节点的特征向量。

在HLBL模型中，给定上下文，下一个词是w的概率，就是由该词编码指定的一串二分决策的概率。因为在一个节点上做出一个决策的概率，只取决于要预测的特征向量（由上下文决定），以及该节点的特征向量，我们可以将下一个词的概率表示成二分决策的概率乘积：

公式三：<img src="http://www.forkosh.com/mathtex.cgi?P(w_{n}=w|w_{1:n-1}=\prodP(d_{i}|q_i,w_{1:n-1})">


di是词汇w的第i位数字编码，qi是编码所对应路径上第i个节点的特征向量。每个决策的概率为：

公式四：<img src="http://www.forkosh.com/mathtex.cgi?P(d_{i}=1|q_{i},w_{1:n-1})=\delta(\hat{r}^Tq_{i}+b_{i})">

其中，<img src="http://www.forkosh.com/mathtex.cgi?\delta(x)">是logistic函数，而<img src="http://www.forkosh.com/mathtex.cgi?\hat{r}">是使用公式一计算得到的要预测的特征向量，在该公式中的bi是该节点的偏置bias，它可以捕获当离开该节点到它的左子节点上的上下文独立的趋势。

<img src="http://www.forkosh.com/mathtex.cgi?P(w_n=w|w_{1:n-1}=\sum_{d\inD(w)}\prod_{i}P(d_i|q_i,w_{1:n-1})">

D(w)表示词汇w所对应的一串编码集合。每个词允许多个编码，可以更好地预测词，具有多种含义或者多种使用模式。每个词使用多种编码，也可以使它更容易将多种不同的词层次结构，组合成单一的结构。这也反映了一个事实：没有单一的层次结构可以表示词之间的所有关系。

使用LBL模型（而非NPLM）来计算局部概率，允许我们避免计算隐层的非线性，这可以让我们的层次模型，比层次化NPLM模型更快作出预测。更重要的是，对于O(log N)个决策中的每一个，层次化NPLM都需要计算一次隐层激活，而HLBL模型只在每次预测时计算一次要预测的特征向量。然而，在LBL模型中，单个二分决策的概率计算的时间复杂性，仍然是特征向量维度D的二次方，这会使高维特征向量的计算开销很高昂。我们可以使时间复杂度与D是线性关系，通过将权重矩阵Ci限制为对角阵。注意，对于上下文size=1的情况，该限制条件不会减小模型表示的幂(power)，我们相信，这种损失(loss)，多于可以通过使用更复杂的树更快的训练时间的补偿。

HLBL模型可以通过最大化（带罚项的）log似然进行训练。因为下一个词的概率只依赖于上下文权重、上下文词汇的特征向量、以及从顶点到词汇所在叶子节点路径上的节点的特征向量，在每个训练case上，只有一小部分（log级别）的参数需要更新。

## 参考

- 1.[A Scalable Hierarchical Distributed Language Model](http://www.cs.toronto.edu/~amnih/papers/hlbl_final.pdf)
- 2.[An empirical study of smoothing techniques for language modeling](http://aclweb.org/anthology/P/P96/P96-1041.pdf)
- 3.[Class-based n-gram models of natural
language](http://www.cs.cmu.edu/~roni/11661/PreviousYearsHandouts/classlm.pdf)
- 4.[A neural probabilistic
language model](http://www.jmlr.org/papers/volume3/bengio03a/bengio03a.pdf)
- 5.[ Quick training of probabilistic neural nets by ´
importance sampling](http://www.iro.umontreal.ca/~lisa/pointeurs/senecal_aistats2003.pdf)
- 6.[Hierarchical probabilistic neural network language model](http://www.iro.umontreal.ca/~lisa/pointeurs/hierarchical-nnlm-aistats05.pdf)
- 7.[Three New Graphical Models for Statistical Language Modelling](https://www.cs.toronto.edu/~amnih/papers/threenew.pdf)
