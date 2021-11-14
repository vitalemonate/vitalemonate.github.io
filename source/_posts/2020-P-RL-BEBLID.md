---
title: 2020 P-RL BEBLID
date: 2021-11-14 10:11:16
tags:
categories: SLAM
mathjax: true
---
# 概述
[BEBLID](http://www.dia.fi.upm.es/~pcr/publications/PRL_2020_web_BEBLID.pdf): Boosted Efficient Binary Local Image Descriptor，是一种高效的基于AdaBoost训练的二进制局部描述子。作者称BEBLID可以达到接近SIFT的精度并且比ORB更高效，目前已经加入了OpenCV4.5.1中。从实验结果来看，BEBLID只有在Hpatches的Patch Verification任务表现比SIFT好，在其余两个任务中性能都不如SIFT，而比ORB更高效是因为BEBLID在实现过程中是并行计算的，但是BEBLID在Hpatches的三个任务中的性能都要好于ORB，所以确实是一个高效且精度较高的二进制描述子。

BEBLID的work flow如下，直观理解就是不像BRIEF只比较测试点对的灰度值来确定描述子，而是**比较测试点对的box区域内的均值(box average)并引用阀值比较的方法来确定描述子**
<div align="center">
<img src="/all_images/2020_P-RL_BEBLID/1.png">
</div>

<!--more-->
# 主要贡献
BEBLID是作者对之前的工作BELID(Boosted Efficient Local Image Descriptor)的改进得到的，主要改进点如下：
* 使用AdaBoost在**不平衡的数据集**上进行训练，以解决实际图像匹配过程中正负样本数量不对称的问题

* 通过最小化一个**所有弱学习器(weak learner,WL)都有相同的权值**的相似度loss来二值化描述子

# 回顾BELID
AdaBoost算法的损失函数为
<div align="center">
<img src="/all_images/2020_P-RL_BEBLID/2.png">
</div>

其中$a_{k}$为第k个WL的权重，$h_{k}$为第k个WL，它依赖于一个feature extraction function(对应下文介绍的**box average function**)以及一个阀值
<div align="center">
<img src="/all_images/2020_P-RL_BEBLID/3.png">
</div>

通过最小化上文的Loss可以得到描述子
<div align="center">
<img src="/all_images/2020_P-RL_BEBLID/4.png">
</div>

# Box average function
整个算法的核心就是f(x)的实现，这一点BEBLID继承了BELID，f(x)计算了两个box内像素均值的差。要高效地实现这个函数，作者使用了积分图的方法，即预先计算图像的积分图，然后即可通过常数次的数学运算得到一个box内的像素均值。
<div align="center">
<img src="/all_images/2020_P-RL_BEBLID/5.png">
</div>

# BEBLID的核心贡献
BEBLID的loss中，各个WL有相同的权值，并且作者将点对之间box average作差的结果与阀值进行比较，将描述子进行二值化，得到了最终的BEBLID
<div align="center">
<img src="/all_images/2020_P-RL_BEBLID/6.png">
</div>

# 实验
在Brown数据集的Liberty Statue patches中训练，所有patches都resize到32×32的大小，box size的集合为{3,5,7,9,11,13,15}，同时将f(x)的值量化为整数，以此来减少阀值集合的大小。

## 确定超参数
### 正负样本的比例
由于在实际的图像匹配或者patch匹配中，正负样本的比例往往是不平衡的。所以作者在训练过程中使用不平衡的正负样本，并在Hpatches上进行评估，评估结果如下。最终后续使用20%的正样本，80%的负样本进行训练。
<div align="center">
<img src="/all_images/2020_P-RL_BEBLID/7.png">
</div>

### 学习率以及描述子位数
<div align="center">
<img src="/all_images/2020_P-RL_BEBLID/8.png">
</div>

最好的学习率为0.0055，描述子位数为256位

## 在Hpatches上与SOTA对比
其中BEBLID的后缀M，V分别表示使用20%，50%的正样本进行训练的结果，256以及512为描述子的位数。
<div align="center">
<img src="/all_images/2020_P-RL_BEBLID/9.png">
</div>

结论：
* Patch Verification任务
  * HardNet+ > BELID > BEBLID > BinBoost > SIFT > others

  * BELID(50%正样本训练)是除了深度学习方法最好的描述子。BEBLID性能不如BELID主要是由于二值化以及BEBLID没有考虑各个WL之间的相关性

  * 尽管BEBLID使用的是简单的基于灰度值差异的方法，但是其性能要高于基于图像梯度的BinBoost以及SIFT。这一点是由于BEBLID中的基于不同size的box average的方法其实可以**在不同尺度近似梯度**

* Image Matching任务: HardNet+ > SIFT > BEBLID-512-M > BELID > others

* Patch Retrieval任务：HardNet+ > SIFT > BELID > others

## 时间对比
作者在Mikolajczyk and Schmid数据集上测试计算描述子的时间，数据集中包含了48张800×640的图像，作者在每张图像中最多提取2000个SURF特征点，然后在三个不同的平台下统计每张图像的平均描述时间，结果如下
<div align="center">
<img src="/all_images/2020_P-RL_BEBLID/10.png">
</div>

结论：
* BEBLID是在各个平台下都是最高效的描述子。分析其算法细节，BEBLID-256应该要比ORB的时间长，但是在由于opencv的ORB实现在计算描述子时没有进行并行实现，而BEBLID使用了并行计算，所以BEBLID比ORB更高效

# 参考
* [BEBLID源码](https://github.com/iago-suarez/BEBLID)
