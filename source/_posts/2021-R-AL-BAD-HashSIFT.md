---
title: 2021 R-AL Revisiting Binary Local Image Description for Resource Limited Devices
date: 2021-11-14 21:55:10
tags:
categories: SLAM
mathjax: true
---
# 概述
[这篇文章](https://arxiv.org/pdf/2108.08380.pdf)与BEBLID出自同一个作者，核心思想是借鉴近年来基于CNN的描述子中的learning techniques，包括triplet ranking loss(TRL), hard negative mining(HNM)以及anchor swapping，来训练传统的基于pixel difference的描述子以及基于image gradient的描述子。作者提出了两个描述子：BAD(Box Average Diference)以及HashSIFT。作者在不同平台以及数据集下对比了两个描述子和常见的描述子，实验结果表明**BAD是当前最快的描述子，HashSIFT是除了基于CNN外的精度最高的描述子**。下图是各个描述子在移动端CPU下的Accuracy vs. computational cost trade-off curve
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
HashSIFT是SIFT的二值化版本，作者二值化SIFT的方法与GCNv2中生成二值化描述子的方法类似。
<div align="center">
<img src="/all_images/2021-R-AL-BAD-HashSIFT/5.png">
</div>

使用随机梯度下降的方法来最小化loss，其中patch triplets的选取与BAD相同，都使用HNM以及anchor swap方法。

# 实验
作者在Liberty数据集上训练BAD以及HashSIFT，在Brown, Oxford以及Hpatches数据集以及一个真实的SfM问题中进行评估。在评估过程中将所有描述子分为三类：基于pixel difference的、基于image gradient的以及基于CNN的。

## 消融实验
* 因为ORB在提取描述子前使用了5×5的box filter进行了平滑(查看opencv 4.5.1源码后发现实际进行的是高斯滤波, box filter被注释掉了)，与BAD的$s=5$，$\theta=0$的box一致，所以选取ORB为baseline。这个baseline在Hpatches的image matching任务中可以达到15.3的mAP

* 各种方法对baseline的提高：将feature selection改为二分类loss，mAP提高了1.35；**引入阀值$\theta$，mAP提高了3.3**；使用TRL，mAP提高1.72；引入尺度，mAP提高了0.28；使用数据增强，mAP提高了0.24

## Brown数据集以及Oxford数据集
其中FPR-95(False Positive Rate at 95% of True Positive Rate)越小越好, mAP越大越好。表中蓝色表示好的指标，绿色表示次好的指标，红色表示第三好的指标。可以看出BAD在基于pixel difference的描述子中表现最好，HashSIFT在基于image gradient的描述子中表现最好（在Oxford数据集中表现次好）
<div align="center">
<img src="/all_images/2021-R-AL-BAD-HashSIFT/6.png">
</div>

## Hpatches数据集
<div align="center">
<img src="/all_images/2021-R-AL-BAD-HashSIFT/7.png">
</div>

结论：
* Patch Verification任务：基于CNN的描述子 > HashSIFT > BAD-512 > BiSIFT > BAD-256 > others

* Image Matching任务：基于CNN的描述子(除了Tfeat-m) > HashSIFT-512 > Tfeat-m > RSIFT > HashSIFT-256 > SIFT > BAD > others

* Patch Retrieval任务：基于CNN的描述子(除了Tfeat-m) > HashSIFT-512 > Tfeat-m > HashSIFT-256 > RSIFT > SIFT > BAD > others

* 总之，在Hpatches上，HashSIFT是除深度学习外的方法中性能最好的，BAD是最高效的，并且BAD相比ORB性能提高了很多

## ETH benchmark
<div align="center">
<img src="/all_images/2021-R-AL-BAD-HashSIFT/8.png">
</div>

结论：
* 在三类方法中，重建结果最好的是基于CNN的描述子

* 在基于pixel difference的描述子中，BAD-256和BAD-512的重建结果最好，且分别可以比ORB重建出多39%和41%的点

* 在基于image gradient的描述子中，HashSIFT重建的sparse points以及observations是最多的，Registered image的数量仅次于RootSIFT，track length次于BinBoost和RootSIFT

## 时间开销
作者在Oxford数据集的每张图像上提取最多2000个特征点，统计计算每张图像描述子的平均时间（表中的括号是执行5次的标准差）
<div align="center">
<img src="/all_images/2021-R-AL-BAD-HashSIFT/9.png">
</div>

结论：
* BAD稍稍领先于BEBLID，是当前最快的描述子

* HashSIFT在准确性和效率之间提供了很好的权衡，并且是基于image gradient的描述子中最快的二进制描述子。这个结论不太准确，从表中的数据来看，HashSIFT在Intel i7以及Snapdragon 855平台下是要比BiSIFT慢的，只有在Exynox Octa S平台下才比BiSIFT快

* HashSIFT比SIFT快的原因是SIFT计算了整张图像的金字塔，而HashSIFT只描述了一个特征点周围32×32的patch，所以在intel i7上比SIFT快6倍，在手机CPU上比SIFT快11倍

## 功耗
<div align="center">
<img src="/all_images/2021-R-AL-BAD-HashSIFT/10.png">
</div>
结论：
* 只有基于pixel difference的描述子才能在SNAPDRAGON 855 CPU上达到实时的效果

* BAD-256的指标与ORB相近，但是速度更快，这一点和BEBLID相同，都是因为在并行计算了描述子

* HashSIFT是基于image gradient的描述子中最节能的

* 基于CNN的描述子的功耗很高，其中CDBin的功耗是BAD-256的146倍，是HashSIFT-256的62倍，在实际应用中仅能提高很小的精度
