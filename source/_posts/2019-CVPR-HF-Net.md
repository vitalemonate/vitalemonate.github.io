---
title: 2019 CVPR HF-Net
date: 2021-11-15 19:56:57
tags:
categories: SLAM
mathjax: true
---
# 概述
[本文](https://arxiv.org/pdf/1812.03506.pdf)出自ETH的自动系统实验室，主要解决的是大尺度情况下的visual localization问题。作者提出一个可以同时预测local feature和global descriptor的网络HF-Net，并将这个网络用于coarse-to-fine的分层视觉定位。核心思路为：先使用global descriptor进行全局检索，得到初步的候选位置，然后在候选中匹配local feature得到最终的定位结果。同时作者使用多任务蒸馏的方式使整个系统可以高效地定位，适用于实时的操作。
<div align="center">
<img src="/all_images/2019 CVPR HF-Net/1.png">
</div>

<!--more-->

# Hierarchical Localization
Hierarchical Localization是本篇文章的前序文章[Leveraging Deep Visual Descriptors for Hierarchical Efficient Localization](https://arxiv.org/pdf/1809.01019.pdf)提出的，整体示意图如下。主要流程为：Prior retrieval, Covisibility clustering以及Local feature matching。
<div align="center">
<img src="/all_images/2019 CVPR HF-Net/2.png">
</div>

## Prior retrieval
在关键帧数据库中使用KNN算法找到K个最相近的关键帧。在前序文章中作者使用的global descriptor是通过NetVLAD蒸馏得到的MobileNetVLAD。

## Covisibility Clustering
将Prior retrieval得到的关键帧根据共视关系进行聚类。如下图中，将(3)中的红圈和绿圈称为place，相当于共视图中的连通区域。
<div align="center">
<img src="/all_images/2019 CVPR HF-Net/3.png">
</div>

## Local matching and Localization
从含有较多关键帧的place开始遍历，首先使用局部描述子匹配place中的3D点与query image的2D关键点，前序文章中使用的局部描述子为SIFT，匹配算法为k-d树。然后使用PnP+RANSAC的方法得到query image的6DoF位姿。如果得到一个位姿就结果算法，否则遍历下一个place。

# HF-Net (Hierarchical Feature Network)
## 网络结构
HF-Net的网络结构如下，backbone为MobileNetv2，decoder部分拆分成local feature和global descriptor两部分，训练时使用SuperPoint作为前者的老师网络，NetVLAD作为后者的老师网络
<div align="center">
<img src="/all_images/2019 CVPR HF-Net/4.png">
</div>

同时作者在附录中给出了网络的具体结构
<div align="center">
<img src="/all_images/2019 CVPR HF-Net/5.png">
</div>

## 训练过程
这部分笔记摘自[参考1](#参考)。作者首先提出训练过程中的2个主要问题：
### 问题1：Data scarcity (数据稀缺性)
对于特征匹配任务来说，很难获取到足够多的同时包含各种场景的全局和局部描述子，以及匹配图像之间的对应groundtruth

### 问题2：Data augmentation (数据增强)
在 SuperPoint 训练过程中使用了大量的数据增强来使得局部特征描述子能够有较强的抗光照、模糊、噪声等等性能。但是过于重的数据增强对于局部特征学习有益的同时却可能对于全局特征的学习造成困难。

### 解决方案：Multi-task distillation (多任务蒸馏)
作者根据之前的一篇文章[Multi-task learning using uncertainty to weigh losses for scene geometry and semantics](https://arxiv.org/pdf/1705.07115.pdf)的技术，通过设计一个**自动学习权重的loss**，一个学生网络S可以同时学习到教师网络$t_1$、$t_2$、$t_3$的知识，而不需要手动调整权重。

据此作者设计了这样的 loss 函数：
<div align="center">
<img src="/all_images/2019 CVPR HF-Net/6.png">
</div>

# 实验
## Local Features Evaluation
由于global descriptor主流的方法就是NetVLAD，不需要进行选择，而目前计算local feature的方法较多，所以本小节实验的目的是给HF-Net的local feature选取一个合适的老师网络。作者在Hpatches和SfM数据集上评估了detector，发现SuperPoint在repeatability以及MLE之间提供了很好的权衡。
<div align="center">
<img src="/all_images/2019 CVPR HF-Net/7.png">
</div>

同时也评估了一些描述子，结果如下
<div align="center">
<img src="/all_images/2019 CVPR HF-Net/8.png">
</div>

得出的结论：
* DOAP在SfM数据集的所有指标都比SuperPoint好，但是由于DOAP是在Hpatches数据集上训练的，所以不能在Hpatches上评估

* Harris/SuperPoint的组合性能较差，这表明**基于学习的描述子需要和对应的detector搭配使用**

* LF-Net和SIFT都是基于多尺度以及patch的描述子，性能不如DOAP和SuperPoint

为了最终在DOAP和SuperPoint中二选一，作者做了另一组实验
<div align="center">
<img src="/all_images/2019 CVPR HF-Net/9.png">
</div>

结果表明：
* 由于DOAP训练的数据集中没有夜晚的场景，所以在夜晚的数据集上性能下降

* 对比 NV+SP 和 NV+HF，可以发现 HF-Net 的 local feature 比 SuperPoint 还略有提升，这一点比较奇怪，因为 HF-Net 本来是从 SuperPoint 蒸馏出来的，按理说性能会有小幅下降才对。对此作者的解释为由于 HF-Net 在同一网络加入了 Global Feature 的监督，使得这一部分的整个网络在更深的语义层次特征中获得了监督，从而变相提升了浅层的 Keypoint 特征

最终作者根据上面的实验综合考虑，HF-Net 使用的是** SuperPoint 提供 Keypoint 和 Descriptor 监督，用 NetVLAD 提供global descriptor监督**。成功的例子示意图如下
<div align="center">
<img src="/all_images/2019 CVPR HF-Net/11.png">
</div>

## Training Dataset
Google Landmarks dataset的185k张图像，主要包括大量白天的城市场景。Berkeley Deep Drive dataset的夜晚和黄昏序列，共37k张图像。

## Large scale Localization
作者在2018CVPR的室外视觉定位benchmark上评估HF-Net的定位性能。评估中用到的数据集为Aachen, RobotCar以及CMU，评估结果如下
<div align="center">
<img src="/all_images/2019 CVPR HF-Net/10.png">
</div>

## Runtime Evaluation
<div align="center">
<img src="/all_images/2019 CVPR HF-Net/12.png">
</div>

#参考
* [论文笔记：From Coarse to Fine: Robust Hierarchical Localization at Large Scale](http://liuxiao.org/2020/07/%e8%ae%ba%e6%96%87%e7%ac%94%e8%ae%b0%ef%bc%9afrom-coarse-to-fine-robust-hierarchical-localization-at-large-scale/)

* [github源码](https://github.com/ethz-asl/hfnet)
