---
layout: post
title: 读懂Transformer
date: 2019-10-31 
description: Transformer结构分析
tags: 机器学习

---  

&emsp;&emsp;Transformer是Google在论文`Attention Is All Your Need`中提出的用于替代RNN提取序列特征的网络结构。在介绍Transformer之前，首先说一下RNN存在的问题。

&emsp;&emsp;RNN常被用于Seq2Seq问题中，用于提取有时间序列的特征，它的两种变体LSTM和GRU则是解决了梯度传播中梯度消失和梯度爆炸的问题，但由于RNN本身的特性，它在计算时必须串行进行，因为每一步的计算都依赖于前一个单元的隐层输出，因此其训练速度较慢，难以加速。

![Fig1 ](/images/paper/transformer-fig1.png)

&emsp;&emsp;基于RNN难以并行化的缺点以及之前提出的基于Attention的做法，Google更进一步，将整个RNN由Transformer结构替代,完全摒弃了RNN的结构，那么Transformer的结构是什么样的呢？它又是如何实现并行化的呢？


## 1. Attention

### 1.1 Self-Attention

&emsp;&emsp;在介绍主角Transformer之前，我们首先介绍一下Self-Attention的结构，因为它是理解Transformer的关键部分。

![Fig2 ](/images/paper/transformer-fig2.png)

&emsp;&emsp;如上图所示，左侧为Bi-RNN的一层，右侧是Self-Attention的一层，我们可以理解为Self-Attention就是RNN结构的替代品，他和RNN一样，输入的一个序列，输出也是一个等长序列，那么它的内部从输入到输出又是如何计算的呢？

![Fig3 ](/images/paper/transformer-fig3.png)

&emsp;&emsp;上图中我们可以把输入看做一个短句子，如`机器学习`，那么x1则是`机`对应的one-hot向量，同理，其它几个xi是另外三个字对应的one-hot向量。那么ai向量则是每个xi词汇对应的词向量，通过W词向量矩阵与xi向量相乘获得，维度多在50-300左右。W词向量矩阵通过上下文学习预训练所得，这里不展开讲。

> 注：此处，我们为了表示方便，将一个字对应一个输入，在实际使用中当然也可以将一个词对应为一个输入，如果以字为输入，那么xi向量的维度则是字的数量【汉字常用字大概为4000左右】，如果以词为输入，则xi向量的长度则是词表的大小。

&emsp;&emsp;之后，每个词向量ai会分别与三个矩阵Wq/Wk和Wv相乘，分别得到三个向量qi/ki/vi.

![Fig4 ](/images/paper/transformer-fig4.png)

&emsp;&emsp;这里的三个矩阵W都是通过网络学习得到，如下图所示，前一层为ai，后一层为$$q_i$$，那么他们之间的权重这是矩阵Wq。

![Fig5 ](/images/paper/transformer-fig5.png)

&emsp;&emsp;在获得了三个向量q、k、v后，我们将要计算每个词与输入中其他所有词的attention，所谓词与词之间的attention，我们可以理解为他们之间的一个关注度，它是一个数值。这个关注度如何计算呢？

![Fig6 ](/images/paper/transformer-fig6.png)

&emsp;&emsp;如上图所示，要计算x1词与其他词之间的attention，则通过x1的查询向量q1分别与自己和其他词的k向量做点积，这样则得到了四个权重。

gongshi

&emsp;&emsp;上式中d是q、k的维度，论文中为64，在这里除以根号d的目的在于防止q和k的维度对于该值的影响，因为q与v的点积随着他们的维度增加会变大，因此除以维度来抵消影响。

![Fig7 ](/images/paper/transformer-fig7.png)

&emsp;&emsp;然后对得到的α向量进行softmax，使得他们的和为1，那么每一个值则对应了x1应该关注的关注度，因此a1的输出b1则是α向量与vi的加权求和：

gongshi

&emsp;&emsp;其余b2/b3/b4同样进行计算。

![Fig8 ](/images/paper/transformer-fig8.png)

&emsp;&emsp;回到上面这张图，我们就解释完了Self-Attention中从输入ai到输出bi的全部计算过程，简单回顾就是：先通过三个矩阵与ai相乘得到对应ai的三个向量qi，ki，vi，然后通过qi分别与所有ki向量点积得到一组attention向量，归一化后再与vi加权求和，得到输出bi向量，这个过程中每个bi之间的计算没有依赖关系，因此可以通过矩阵计算一同完成，而GPU对于矩阵运算的加速非常擅长，因此可以大大提升计算速度。

### 1.2 Multi-Head Attention

&emsp;&emsp;上面讲的Self-Attention是是单个head的，他们有一组Wq、Wk和Wv，那么multi-head我们可以理解为多组矩阵Wq，Wk和Wv，下图展示了2个head的情况：

![Fig9 ](/images/paper/transformer-fig9.png)

&emsp;&emsp;每个输入ai，通过上述计算可以得到两个bi，两个bi进行concat，然后可以再乘上一个权重矩阵得到与qi同维度的向量，这个权重矩阵和上述过程中Wq一样，也是通过网络学习得到。

&emsp;&emsp;那么为什么要使用多个head呢？每个head又有什么不同呢？

&emsp;&emsp;论文给出的解释是不同的head的关注范围会不一样，比如一个只关注附近的词汇，而另一个更关注全局所有词汇，论文后面所附可视化的展示也证实了这一点。

&emsp;&emsp;OK，到此为止，终于把Self-Attention的部分讲完了，下面我们来看一下Transformer吧。

### 1.3 

## 2. Transformer

### 2.1 Transformer结构

![Fig10 ](/images/paper/transformer-fig10.jpg)

&emsp;&emsp;相信这个图大家已经在各种教程上看过无数遍了，今天我们再仔细理一下这其中的细节。

### 2.2 位置编码

&emsp;&emsp;在上图中的输入和输出Embedding时都有位置编码信息加入，在这里加入位置编码是由于Self_attention内部对于每个词的位置实际上是没有信息的，词的位置对Self-Attention的计算结果没有影响，这就导致Transformer结构无法捕捉位置信息，所以在最开始的Embedding与位置编码相加，这里不是concat，不使用concat原因下图解释。

![Fig11 ](/images/paper/transformer-fig11.png)

### 2.3 