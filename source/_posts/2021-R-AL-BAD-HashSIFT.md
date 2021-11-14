---
title: 2021 R-AL Revisiting Binary Local Image Description for Resource Limited Devices
date: 2021-11-14 21:55:10
tags:
categories: SLAM
mathjax: true
---
# 概述
这篇文章与BEBLID出自同一个作者，核心思想是借鉴近年来基于CNN的描述子中的learning techniques，包括triplet ranking loss(TRL), hard negative mining(HNM)以及anchor swapping，来训练传统的基于pixel difference描述子以及基于image gradient的描述子。作者提出了两个描述子：BAD(Box Average Diference)以及HashSIFT。作者在不同平台以及数据集下对比了两个描述子和常见的描述子，实验结果表明**BAD是当前最快的描述子，HashSIFT是除了基于CNN外的精度最高的描述子**。下图是各个描述子在移动端CPU下的Accuracy vs. computational cost trade-off curve
<div align="center">
<img src="/all_images/2021-R-AL-BAD-HashSIFT/1.png">
</div>

<!--more-->
# BAD descriptor
BAD是由BEBLID改进而来，作者使用贪心的策略选取点对(feature)并使用TRL，HNM以及anchor swapping改进BEBLID的训练过程。相关参数说明：
* feature extraction function: $f(x)$
* binary weak-descriptor: $h(x;f,\theta)=\left \{+1\ if\ f(x)\le\theta;-1\ otherwise  \right \}$
* selected weak-descriptors: $h(x)=\left [ h_1(x),h_2(x),...h_K(x) \right ] ^T$
* similarity between the binary descriptors of patches $x$ and $y$: $S(x; y) = h(x)^Th(y)$

作者引入triplets loss并使用增量式的方法来最小化triplets loss:
<div align="center">
<img src="/all_images/2021-R-AL-BAD-HashSIFT/2.png">
</div>

其中Feature selection algorithm如下
<div align="center">
<img src="/all_images/2021-R-AL-BAD-HashSIFT/3.png">
</div>

sampleTriplets函数使用了HNM和anchor swap方法。每轮迭代中随机采样1000个任意尺度的候选feature，从中选取能最小化的loss的集合。确定阀值的算法为
<div align="center">
<img src="/all_images/2021-R-AL-BAD-HashSIFT/4.png">
</div>

# HashSIFT
HashSIFT是SIFT的二进制版本，作者二值化SIFT的方法与GCNv2中生成二值化描述子的方法类似。
<div align="center">
<img src="/all_images/2021-R-AL-BAD-HashSIFT/5.png">
</div>

使用随机梯度下降的方法来最小化loss，其中patch triplets的选取与BAD相同，都使用HNM以及anchor swap方法
