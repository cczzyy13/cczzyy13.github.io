---
title: "TensorFlow实战"
categories:
- 算法
tags:
- 学习笔记
toc: true

---

{% include lib/mathjax.html %}



# 深度学习简介

卡内基梅隆大学的Tom Michael Mitchell 教授在1997年出版的书籍 *MachineLearning* 中对机器学习的定义：如果一个程序可以在任务T上，随着经验E的增加，效果P也可以随之增加，则称这个程序可以从经验中学习。

深度学习解决的核心问题之一就是自动地将简单的特征组合成更加复杂的特征，并使用这些特征解决问题。

## 深度学习应用

- 计算机视觉：无人驾驶、谷歌地图、以图搜图、美颜相机、人脸识别进行风控（贷款发放）、光学字符识别（邮政编码识别、银行支票数额识别、谷歌街景门牌号识别、扫描图书数字化实现内容搜索）等
- 语音识别：智能语音系统（Siri系统）、语音搜索、同声传译等
- 自然语言处理：机器翻译、情感分析等
- 人机博弈：象棋、围棋、即时战略游戏（星际争霸）等

## 深度学习工具介绍与对比

TensorFlow 是谷歌于2015年11月9日正式开源的计算框架。

![](https://ws4.sinaimg.cn/large/006tNbRwgy1fuksfiu1hhj31600qgau1.jpg)



# TensorFlow 环境搭建

TensorFlow 主要依赖包：

- Protocol Buffer，结构数据序列化的工具，TensorFlow 中大部分数据都是通过 Protocol Buffer 的形式储存的。
- Bazel，谷歌开源的编译工具，谷歌官方给出的大部分样例程序都是通过 Bazel 编译的。

TensorFlow 安装：

1. 使用Docker安装
2. 使用pip安装 ，地址：https://storage.googleapis.com/tensorflow/mac/cpu/tensorflow-1.4.0-py3-none-any.whl	
3. 从源代码编译安装

TensorFlow 测试样例：

```
import tensorflow as tf
a = tf.constant([1.0, 2.0], name = "a")
b = tf.constant([2.0, 3.0], name = "b")
result = a + b
sess = tf.Session()  #生成一个会话
sess.run(result)
```



# TensorFlow 入门

## TensorFlow 计算模型——计算图

Tensorflow 是一个通过计算图的形式来表述计算的编程系统。TensorFlow 中的每一个计算都是计算图上的一个节点，而节点之间的边描述了计算之间的依赖关系。

通过`tf.get_default_graph`函数可以获取当前默认的计算图。

通过 `tf.Graph`函数来生成新的计算图。

### 张量

张量可以被简单理解为多维数组，但它在TensorFlow中的实现并不是直接采用数组的形式。在张量中并没有真正保存数字，它保存的是如何得到这些数字的计算过程。

一个张量中主要保存了三个属性：名字、维度和类型。

- 名字：是一个张量的唯一标识符，也给出了张量是如何计算出来的。
- 维度：描述了一个张量的维度信息。
- 类型：每个张量会有一个唯一的类型，TensorFlow会对参与运算的所有张量进行类型的检查，当发现类型不匹配时会报错。

### 会话

会话拥有并管理 TensorFlow 程序运行时的所有资源。TensorFlow 中使用会话的模式一般有两种：

- 需要明确调用会话生成函数和关闭会话函数

  ```
  sess = tf.Session()  #创建一个会话
  sess.run()  #得到张量的运算结果
  sess.close()	#关闭会话释放资源
  ```


- 使用python的上下文管理器

  ```
  with tf.Session() as sess:
  	sess.run()
  #无需调用close函数来关闭会话，当上下文退出时会话关闭和资源释放都自动完成
  ```

TensorFlow 不糊自动生成默认的会话，需要手动指定。当默认的会话被指定以后可以通过 `tf.Tensor.eval` 函数来计算一个张量的取值。

```
sess = tf.Session()
with sess.as_default():
	print(result.eval())
##以下代码可完成相同的功能
sess = tf.Sesion()
print(sess.run(result))
print(result.eval(session=sess))
```

在交互式环境下（Python脚本或jupyter的编辑器），可使用 `tf.InteractiveSession` 函数来自动将生成的会话注册为默认会话。

```
sess = tf.InteractiveSession()
print(result.eval())
sess.close()
```






























​				
​			
​		
​	