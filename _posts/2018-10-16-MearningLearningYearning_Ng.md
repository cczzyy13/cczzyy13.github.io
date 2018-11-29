---
title: "MachineLearningYearning_ANDREW NG"
categories:
- 机器学习
tags:
- 学习笔记
toc: true
---

{% include lib/mathjax.html %}



## 一、设置 dev/test sets

定义：

- Training set: 用来跑算法
- Development set: 用于调参数、选特征等
- Test set: 用于评估算法的表现，但不要用于选择模型算法或者参数

技巧：

- 选择你期望在未来得到或者想要在上面表现好的数据分布的 dev/test sets ，这有可能与training data 的分布不同
- dev/test sets 需要来自于同一个分布
- 选择一个单一的数值评价指标来优化。如果有N个目标需要考虑，把它们结合成单一的公式（例如2/F1=1/Precision + 1/Recall）或者将N-1个设置为要满足的条件然后优化剩余那个
- 尽可能快的设置好dev/test sets 以及度量指标，例如一周之内
- dev set 应该足够大到可以侦测到算法精确度的变化，但不要太大。test set 应该大到让你可以 confident 地评估你的系统最终表现
- 启发式的 70%/30% 的 train/test 比例在数据量很大的问题上不适用。dev 和 test sets 可以比 30% 小很多
- 如果dev set 和 度量不再将你的团队引向正确的方向，赶紧改变它们。
  1. 如果 dev set 过拟合，收集更多的 dev set 数据
  2. 如果你关心的数据的真实分布和 dev/test  set 的分布不同，更换新的 dev/test set 数据
  3. 如果度量指标不再能衡量你认为重要的指标，改变度量指标



## 二、基本的错误分析

定义：

- 错误分析(Error analysis) : 指的是检验 dev set 中算法分错类别的样本的过程

技巧：

- 给算法分错类的样本们一些基本的描述评论以及手动的将它们分成一些类别，得到类似如下的图，这是一个持续迭代的过程，然后就可以了解在哪一种分类错误上花费精力最多可以消除多少的错误

  ![](https://ws3.sinaimg.cn/large/006tNbRwgy1fwb6c9i5yhj311c0dkgn2.jpg)

- 清洗 dev/test sets 中 label 错误的数据，当它们占据的错误分类样本的比例足够大时，给它们重新打上正确的标签，并对所有的 dev/test sets 数据检查标签是否正确。对于学术研究论文来说这很重要来保证 test set 精确度的无偏估计，对于一些产品应用来说，这些小偏差是可以接受的

- 当 dev set 足够大的时候，可以将它分为两个子集

  - Eyeball dev set : 进行错误分析
  - Blackbox dev set : 用于选择算法以及调节超参

  如果Eyeball dev set 上的表现比Blackbox dev set 上好太多，说明模型对 Eyeball dev set 过拟合，这个时候应该设立新的Eyeball dev set

- 当数据量太小时，可以选择不分割，因为两个集合的样本数需要足够达到它们存在的目的。经验来说

  - 20～50个错误样本可以较好的反应错误样本的主要情况
  - 100个以上错误样本便可以十分好的反应
  - 很少有人分析超过500～1000个的错误样本
  - 根据分类器的错误率可以相应计算出所需的 Eyeball dev set 的数量

  对于一些人类也做不好的分类任务，Eyeball dev set 存在的意义也不大，因为人类也无法分清算法分类错误的原因



## 三、偏差(Bias)和方差(Variance)

定义：

- 算法的偏差(Bias) : 指的是算法在 training set 上的错误率
- 算法的方差(Variance) : 指的是算法在 dev(or test) set 上比 training set  上差多少
- 最佳错误率(Optimal error rate) : ("unavoidable bias")一个完美的分类器能到达的最好的成绩(错误率)
- 可避免偏差(Avoidable bias) : 指的是算法比完美的分类器在 training set 上表现差多少

Optimal error rate 也被称作 Bayes error rate.

Bias = Optimal error rate + Avoidable bias.

降低 Avoidable bias 的技巧：

- 增加模型的大小(例如神经网络的数量或层数) : 这种方法有可能会增加方差，可以通过增加正则项的方式降低方差
- 根据error analysis的结果改变输入特征 : 增加一些额外的特征算法消除特定类别的错误。理论上，增加更多的特征也有可能会增加方差，类似，也可以使用正则项的方法消除方差的增加
- 减小或消除正则化(L2, L1, dropout) : 这会减小 avoidable bias，但是增加方差
- 改变模型结构 : 选择更适合问题的模型结构
- 在training data上做error analysis

降低 variance 的技巧：

- 增加更多的训练数据 : 如果你有渠道获得更多的数据以及充足的计算力来获得计算数据，这是最简单并且最可信的降低方差的方法
- 增加正则项 : 这会降低方差但是增加方差
- 增加 early stopping : 例如更早的停止梯度下降迭代，这个技巧会降低方差但是增加偏差，有点像是正则项技巧
- 特征选择来减少输入特征的数量或类型
- 降低模型大小



## 四、学习曲线

定义：

- 学习曲线(Learning curve) : dev set 错误率随 training set 大小变化的曲线
- Desired error rate : 我们期望学习算法最终会达到的目标错误率

- Training error curve : dev set 错误率随 training set 大小变化的曲线，通常training set 越大 training set error 越大 

  ![](https://ws4.sinaimg.cn/large/006tNbRwgy1fwh1mf8h9dj30t80dqq74.jpg)

技巧：

- 通过观察三根线之间的关系趋势来判断，增加training set size 能否达到目标
- 当 training set 十分小或者类别特别多或者正负样本比例相差特别大的时候，曲线可能会不太稳定，难以看出趋势，可以通过如下方法解决：
  - 多次随机有放回取样(sampling with replacement)，计算 training/dev set error的均值来画图
  - 如果正负样本比例相差很大或者有很多个类别，可以分层取样，确保样本中每个类别中的比例与整个原始 training set 中的比例接近或一样

## 五、与人类级别的表现比较

