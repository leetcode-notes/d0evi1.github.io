---
layout: post
title: Candidate Sampling介绍
description: 
modified: 2016-07-03
tags: 
---

我们有一个multi-class或者multi-label问题，其中每个训练样本$$(x_i, T_i)$$包含了一个上下文$$x_i$$，以及一个关于target classese $$T_i$$的小集合，它在一个关于可能classes的大型空间L的范围之外。例如，我们的问题可能是，给定前面的词汇，预测在句子中的下一个词。

我们希望学习到一个兼容函数（compatibility function） $$F(x,y)$$，它会说明关于某个class y以及一个context x间的兼容性。例如——给定该上下文，该class的概率。

“穷举（Exhaustive）”训练法，比如：softmax和logistic regression需要我们为每个训练样本，每个类$$y \in L$$去计算F(x,y)。当$$\mid L \mid$$很大时，计算开销会很大。

“候选采样（Candidate Sampling）”训练法，涉及到构建这样一个训练任务：对于每个训练样本$$(x_i, T_i)$$，我们只需要为候选类$$C_i \subset L$$评估F(x,y)。通常，候选集合$$C_i$$是target classes的union，它会随机选择对（其它）classes $$S_i \subset L$$进行抽样。

$$
C_i = T_i \cap S_i
$$

随机样本$$S_i$$可能或不可能依赖于$$x_i$$和/或$$T_i$$。

训练算法会采用神经网络的形式，其中表示F(x,y)的layer会通过BP算法从一个loss function中进行训练。

图1

- $$Q(y \mid x)$$: 被定义为：给定context x，在抽样classes的集合中，根据抽样算法得到class y的概率（或：期望count）。
- $$K(x)$$：是一个任意函数（arbitrary function），不依赖于候选类（candidate class）。由于softmax涉及到一个归一化（normalization），对这种函数的求和不会影响到计算概率。
- logistic training loss= $$\sum\limits_i (\sum )$$
- softmax training loss = $$\sum\limits_i (-G(x_i,t_i) + log(\sum\limits_{y \in POS_i \cap NEG_i}) exp(G(x_i,y))))$$
- NCE 和Negatvie Sampling可以泛化到$$T_i$$是一个multiset的情况。在这种情况中，$$P(y \mid x)$$表示在$$T_i$$中y的期望数（expected count）。相似的，NCE，Negative Sampling和Sampled Logistic可以泛化到$$S_i$$是一个multiset的情况。在这种情况下，$$Q(y \mid x)$$表示在$$S_i$$中y的期望数（expected count）。

# Sampled Softmax

参考：[http://arxiv.org/abs/1412.2007](http://arxiv.org/abs/1412.2007)

假设我们有一个单标签问题（single-label）。每个训练样本$$(x_i, \lbrace t_i \rbrace)$$包含了一个context以及一个target class。我们将$$P(y \mid x)$$作为：给定context x下，一个target class y的概率。

我们可以训练一个函数F(x,y)来生成softmax logits——也就是说，给定context，该class相对log概率：

$$
F(x,y) \leftarrow log(P(y|x)) + K(x)
$$

其中，K(x)是一个任意函数，它不依赖于y。

在full softmax训练中，对于每个训练样本$$(x_i,\lbrace t_i \rbrace)$$，我们会为在$$y \in L$$中的所有类计算logits $$F(x_i,y)$$。**如果类L很大，计算很会昂贵**。

在"Sampled Softmax"中，对于每个训练样本$$(x_i, \lbrace t_i \rbrace)$$，**我们会根据一个选择抽样函数$$Q(y \mid x)$$来选择一个关于“sampled” classese的小集合$$S_i \subset L$$**。每个被包含在$$S_i$$中的类，它与概率$$Q(y \mid x_i)$$完全独立。

$$
P(S_i = S|x_i) = \prod_{y \in S} Q(y|x_i) \prod_{y \in (L-S)} (1-Q(y|x_i))
$$

我们创建一个候选集合$$C_i$$，它包含了关于target class和sampled classes的union：

$$
C_i = S_i \cup \lbrace t_i \rbrace
$$

我们的训练任务是为了指出：**在给定集合$$C_i$$上，在$$C_i$$中哪个类是target class**。

对于每个类$$y \in C_i$$，给定我们的先验$$x_i$$和$$C_i$$，我们希望计算target class y的后验概率。

使用Bayes' rule：

[bayes]{https://math.stackexchange.com/questions/549887/bayes-theorem-with-multiple-random-variables}

$$
P(Z|X,Y) = P(Y,Z|X) P(X) / P(X,Y) = P(Y,Z|X) P(Y|X)
$$...(b)

得到：

$$
P(t_i=y|x_i,C_i) = P(t_i=y,C_i|x_i) / P(C_i|x_i) \\
=P(t_i=y|x_i) P(C_i|t_i=y,x_i) / P(C_i|x_i) \\
=P(y|x_i)P(C_i|t_i=y,x_i) / P(C_i|x_i)
$$

接着，为了计算$$P(C_i \mid t_i=y,x_i)$$，我们注意到为了让它发生，$$S_i$$可以包含y或也可以不包含y，但必须包含$$C_i$$所有其它元素，并且必须不包含在$$C_i$$任意classes。因此：

$$
P(t_i=y|x_i, C_i) = P(y|x_i) \prod_{y \in C_i - \lbrace y \rbrace} Q({y'}|x_i) \prod_{y \in (L-C_i)} (1-Q({y'}|x_i)) / P(C_i | x_i) \\
= \frac{P(y|x_i)}{Q(y|x_i)} \prod_{ {y'} \in C_i} Q({y'}|x_i) \prod_{ {y'} \in (L-C_i)} (1-Q({y'}|x_i))/P(C_i|x_i) \\
=\frac{P(y|x_i)}{Q(y|x_i)} / K(x_i,C_i)
$$

其中，$$K(x_i,C_i)$$是一个与y无关的函数。因而：

$$
log(P(t_i=y | x_i, C_i)) = log(P(y|x_i)) - log(Q(y|x_i)) + {K'} (x_i,C_i)
$$

这些是relative logits，应feed给一个softmax classifier，来预测在$$C_i$$中的哪个candidates是正样本（true）。

因此，我们尝试训练函数F(x,y)来逼近$$log(P(y \mid x))$$，它会采用在我们的网络中表示F(x,y)的layer，减去$$log(Q(y \mid x))$$，然后将结果传给一个softmax classifier来预测哪个candidate是true样本。

$$
training-softmax-input = F(x,y) - log(Q(y|x))
$$

从该classifer对梯度进行BP，可以训练任何我们想到的F。

# 参考

[https://www.tensorflow.org/extras/candidate_sampling.pdf](https://www.tensorflow.org/extras/candidate_sampling.pdf)
