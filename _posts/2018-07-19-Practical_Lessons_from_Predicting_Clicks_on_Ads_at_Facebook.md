---
title: "Practical_Lessons_from_Predicting_Clicks_on_Ads_at_Facebook 论文笔记 "
categories:
- 机器学习
tags:
- 算法
---

{% include lib/mathjax.html %}

[论文链接](https://github.com/cczzyy13/book/blob/master/Practical_Lessons_from_Predicting_Clicks_on_Ads_at_Facebook.pdf)

## 一、简介

这篇文章是工业界点击率预估方向的一篇重要论文（Facebook）。文章提出 GBDT+LR 的混合模型，使用 GBDT 对特征值进行加工转换，再使用 LR 线性分类器对转换后的特征进行拟合。

## 二、实验设置

文章选取 2013 年四季度随机的一周的数据作为数据集，并提出 "online joiner" 系统将离线数据转化为在线数据。

文章的评估度量是 NE（Normalized Entropy）和 Calibration。

首先定义 CTR（click through rate）为点击转化率。

NE 的定义是 样本CTR标准化后的预测对数损失。\\(y_i\\)是lable，\\(p_i\\)为估计的点击概率，\\(p\\)为训练样本平均的CTR，NE 的计算公式是

![](https://ws2.sinaimg.cn/large/006tKfTcgy1ftfflq8bftj30pw05sjrx.jpg)

NE 越小，模型的预测效果越好。

Calibration 定义为平均预估CTR与经验平均CTR的比值，也就是预测的点击数量比上实际观测到的点击量。

Calibration越接近1，模型越好。

## 三、模型结构

Boosted decision trees  串联一个  probabilistic sparse linear classifier.

![](https://ws2.sinaimg.cn/large/006tKfTcgy1ftffwmznxbj313e0pkjwx.jpg)

Boosted decision trees 将特征值转化为稀疏的01向量（one-hot），假设模型产生了 n 棵树，特征转化后得到的结构向量为 \\(x = (e_{i_1},\cdots,e_{i_n})\\)，其中 \\(e_i\\) 为第 i 个单位向量，表示第 i 棵树中样本被分类到的哪一片叶子。假如模型有两棵子树，子树的叶子分别为3和2，样本在两棵树中分别被分到了第2片和第1片叶子，则转化后得到的向量为 [0,1,0,1,0]。

对于有标签的广告样本 \\((x,y)\\)，定义 \\(s(y,x,w) = y w^T x  = y \sum_{j=1}^{n} w_{j,i_j}\\)。

可以使用以下两种办法构建线性分类器：BOPR 和 SGD+LR。

**BOPR（Bayesian online learning scheme for probit regression）**: 模型为

![](https://ws4.sinaimg.cn/large/006tKfTcgy1ftfgeeb4q4j30ti07m754.jpg)

通过以下迭代得到 \\(w\\) 的后验分布的均值和方差。（函数 v 和 w 文章中定义好）

![](https://ws4.sinaimg.cn/large/006tKfTcgy1ftfgj5nb06j30rw0b0abc.jpg)

**LR + SGD** 模型：似然函数为

![](https://ws1.sinaimg.cn/large/006tKfTcgy1ftfgmateb3j30tc02q74h.jpg)

SGD（Stochastic Gradient Descent）算法的解该问题的迭代公式为：

![](https://ws3.sinaimg.cn/large/006tKfTcgy1ftfgnnmaymj30u005kdgw.jpg)

### 决策树特征转化

一般来说有两种简单的方法转化得到线性分类器的输入特征：

1、对于连续特征，转化为二进制，并学习好有效的二进制边界。

2、对于类别特征，使用笛卡尔积（文中提到了k-d树）。

本文使用 [Gradient Boosting Machine](https://github.com/cczzyy13/book/blob/master/Greedy_function_approximation_a_gradient_boosting_machine.pdf) 中的经典 \\(L_2 - \\) TreeBoost algorithm。实验比较了LR only ，Trees only 以及 LR + Trees 的结果：

![](https://ws4.sinaimg.cn/large/006tKfTcgy1ftfgyrvbnrj30xq0degoh.jpg)

### 数据新鲜度

实验证明数据越新鲜，模型的预测结果越好，因此，可以对 GBDT 模型部分进行日级别的训练更新，对Linear classifier部分，进行 online real-time 的训练。

### 在线线性分类器

对于 LR + SGD 方法的学习率参数部分，比较了以下5种选择：

![](https://ws1.sinaimg.cn/large/006tKfTcgy1ftfh3xxfrpj30vs0wojvf.jpg)

实验后发现第一种的预测精确度最好（即 SGD with per-coordinate learning rate），而这种方法的效果与BOPR 几乎一样。

**模型比较：**

LR 模型参数更少，缓存更优计算更快。

BOPR 模型预测了点击概率的整体分布，得到的结果更多。

## 四、Online data joiner

这个系统可以生成 real-time 的训练数据流来进行 online learning。

系统确定好合适的等待窗口时间（等待多久之后视为 "no click"），来平衡好 recency 和 click coverage。并通过 HashQueue 的方式来形成数据流。整个模型的训练过程如下：

![](https://ws1.sinaimg.cn/large/006tKfTcgy1ftfhf03a2gj30vo0jujtu.jpg)

## 五、Containing memory and latency

### 决策树的数量

![](https://ws2.sinaimg.cn/large/006tKfTcgy1ftfhj4wv7uj30yc0p2wih.jpg)

大部分的 NE 由前 500 棵树贡献。

### 特征的重要性

特征分为历史特征和情境特征。

历史特征是指广告或者用户前期的交互表现，例如广告上一周的点击转化率或者用户的平均点击转化率。

情境特征是指广告要被投放的当前的情境信息，例如用户正在使用的服务或者网站。

![](https://ws2.sinaimg.cn/large/006tKfTcgy1ftfhpqea87j30w60qyae9.jpg)

实验表明历史特征比情境特征的贡献更大，但是对于cold start(新用户或者新广告)来说，情境特征是不可或缺的。

## 大规模数据的取样

### uniform subsampling

对正负样本使用相同的取样率，对于不同的取样率，实验结果如下：

![](https://ws4.sinaimg.cn/large/006tKfTcgy1ftfhtxrag3j30ye0tm7d1.jpg)

大约 10% 的数据便可贡献大部分的 NE。

### negative down sampling

对负样本进行降采样，实验表明负样本降采样对模型训练的效果提升较大。可以选取一个合适的降采样率。

### 模型校准

由于采样方式会带来偏差，需要对预测结果进行纠正校准：

假设 p 是降采样空间的预测值， w 是降采样率，则校准之后的预测值为

$$ q = \frac{p}{p+(1-p)/w}.$$

