---
title: GCNv2系列
date: 2021-11-06 15:33:41
tags:
categories: SLAM
---
# 概述
GCN系列共两篇文章，GCNv1为2018年发表在ICRA上的[Geometric Correspondence Network for Camera Motion Estimation](https://ieeexplore.ieee.org/document/8260906/), GCNv2为发表在R-AL上的[GCNv2: Efficient Correspondence Prediction for Real-Time SLAM](https://arxiv.org/pdf/1902.11046.pdf)。二者都是基于CNN的提取特征点和局部描述符的网络。

# GCNv1
GCNv1的网络结构为CNN+RNN，训练的思想为**Learning by warping**: 借助刚体变换，通过将source frame中的点warp到reference frame中实现对网络的优化

## Learning by warping
作者在Introduction中有下面一段话:
>To train the network we extract high gradient points in one image and warp them using the camera motion to the next image. It is important to note that we do not try to learn to reproduce the high gradient points or any standard descriptors based on
them, but rather the network learns to find features and descriptors that can be matched well. In fact, the points in the warped
image may not even be as detectable based on the gradient as they were in the unwarped image

网络学习的重点在于找到那些**适合匹配的特征点**，而不是普通意义上的那些**hight gradient points**。作者先在一幅图像中提取high gradient points，然后通过warping找到这些点在另一幅图像中的对应点。事实上这些warp到新图像中的点也许并不是该图像上的hight gradient points。

## 网络结构
<div align="center">
<img src="/home/wanghao/blogs/source/all_images/GCN系列/1.png">
</div>

整个网络可以分为两部分：CNN+RCNN。CNN的结构为ResNet-50 + Deconvolution + shortcut的形式结构

### 特征关联
所谓特征关联，其实就是选出正确的特征点，在两张图片中，他们之间的相对位姿已经由ground-truth提供了，所以如果一张图片中的一个特征点能够按照相对位姿投影到另一张图片中对应的特征点上，那么这就是一对正样本，否则就是负样本。具体实现的方式使用的是深度度量学习（Deep **Metric Learning**），核心在于损失函数**triplet loss**的使用


# GCNv2

# 参考
* [论文翻译 GCN-v1:Geometric Correspondence Network for Camera Motion Estimation](https://www.pianshen.com/article/61111891744/)

* [语义SLAM | 深度学习用于特征提取 : GCN-SLAM（一）](https://zhuanlan.zhihu.com/p/68026746)
