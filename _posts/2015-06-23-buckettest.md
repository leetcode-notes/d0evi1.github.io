---
layout: post
title: 浅谈分桶测试 
description: 
modified: 2013-07-03
tags: [分桶测试 bucket 推荐系统]
---

在推荐系统中，用于测试模型性能，通常会选定随机选定部分用户，观察这些用户在推荐项上的行为。这就是我们常说的分桶测试（bucket tests）。

假定有两个推荐模型：模型A和模型B。我们可以创建两个不相交的样本：基于用户（用户id）的样本选择方式创建、或基于请求（用户访问行为）的样本选择方式创建。接着，对于第一个样本，使用模型A； 对于第二个样本，使用模型B。并持续服务一段时间。这里的每个样本，称为一个桶（bucket）。通常有两种常用的分桶方式：

- 1.**基于用户的分桶（User-based bucket）**：这样的桶，是一个随机选定用户的集合。一种简单的方式是，使用一个hash函数，为每个user id生成一个hash值，选择一个特定的范围指向一个桶。例如：Ron Rivest设计的md5。

- 2.**基于请求的分桶（Request-based bucket）**：这样的桶，是一个随机选择的请求的集合。常用的做法是，为每个请求生成一个随机数，然后将对应指定范围的请求随机数指定到某个桶内。注意，在这样的桶中，在实验期间，同一个用户不同的访问，有可能属于不同的分桶。

基于用户的分桶，通常比基于请求的分桶更简洁、更独立。例如，当使用基于请求的分桶时，一个用户使用模型A的响应（Response），可能会影响到模型B。但是，在基于用户的分桶中，这个现象不会发生。另外，任何长期用户行为都可以在基于用户的分桶中进行。然而，如果在基于用户的分桶中使用一个简单模型，该分桶的用户可能会收到不好的结果，这样也会导致较差的用户体验。而基于请求的分桶则对这种模型相对不敏感些，因为一个用户的所有请求不一样分配到相同的bucket中。总之，基于用户的分桶更受欢迎些。

在受控的实验中，分桶的所有设置应该一致，除了为每个分桶分配的模型不同；模型A用于服务分桶1；模型B用于服务分桶2。特别的，对于两个分桶来说，我们要使用相同的选择方式准则。例如，某一个分桶只包含登陆用户，那么另一个分桶也必须一致。

当使用基于用户的分桶时，**对于不同的测试，最好使用独立的各不相同的hash函数，以保持正交性**。例如，假设我们在一个web页面具有两个推荐模块，每个模块对应两个要测试的模型。两个对应的测试模块：test1和test2。对于每个test i，都有两个对应的推荐模型：Ai和Bi。如果我们在两组test上使用相同的hash函数为用户分配hash值，hash值低于某个阀值的使用模型Ai，剩下的使用模型Bi，这样，模型A1的用户与模型A2的用户相同；模型B1的用户与模型B2的用户相同。由于涉及到与A2和B2的交互，这会导致模型A1与模型B1之间的比较不够合理。解决这种问题的一个方法是，确保分配给A1模型的用户概率与test2中的A2或B2模型相互独立。这很容易实现，如果我们在test1中使用的将user id映射后的hash值与test2中相互统计独立即可。使用独立的hash函数，可以帮助我们控制当前测试与之前测试的独立性。

另一个有用的实践是，使用相同的模型服务两组分桶，并确认两个桶对应的性能指标是否统计上相似。这样的测试通常称为**A/A test**。它不仅为继承的统计变量提供了一个好的估计，而且还可以帮助在实验阶段发现明显错误。另一个有用的实践是，运行一个分桶测试至少需要一到两周，因为，用户行为通常有一周时间里有时间周期性上的不同。当一个新的推荐模型推荐对应在其它模型上完全不同的item时，由于**新奇性效应（novelty effect），用户(user)可能倾向于在初始阶段点击更积极**。为了减小因此造成的潜在偏差，当监控测试指标时，通常抛弃开始阶段的测试结果是很有用的。

**标准的实验设计方法可以用来决定一个分桶所需要size以达到统计显著性（statistical significance）**。拔靴法（Bootstrap sampling）在决定性能指标的方差时很管用，它可以用来帮助计算分桶的抽样size。详见：Montgomery(2012)、Efron和Tibshirani(1993).

参考：

- [Statistical Methods for Recommender Systems](https://books.google.com/books?id=bCZ0CwAAQBAJ&pg=PT81&lpg=PT81&dq=recommend+system+bucket+test+hash&source=bl&ots=dIrqWpBtGl&sig=MsEAAAE3Na7IavjgLzneWZF8nxU&hl=zh-CN&sa=X&ved=0ahUKEwiu4a-5lPPNAhXMbZoKHYTNCp0Q6AEIHjAA#v=onepage&q=recommend%20system%20bucket%20test%20hash&f=false)
