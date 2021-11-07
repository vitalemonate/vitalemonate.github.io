---
title: GCNv2系列
date: 2021-11-06 15:33:41
tags:
categories: SLAM
---
# 概述
GCN系列共两篇文章，GCNv1为2018年发表在ICRA上的[Geometric Correspondence Network for Camera Motion Estimation](https://ieeexplore.ieee.org/document/8260906/), GCNv2为发表在R-AL上的[GCNv2: Efficient Correspondence Prediction for Real-Time SLAM](https://arxiv.org/pdf/1902.11046.pdf)。二者都是基于CNN的提取特征点和局部描述符的网络。
<!--more-->
# GCNv1
GCNv1的网络结构为CNN+RNN，训练的思想为**Learning by warping**: 借助刚体变换，通过将source frame中的点warp到reference frame中实现对网络的优化

## Learning by warping
作者在Introduction中有下面一段话:
>To train the network we extract high gradient points in one image and warp them using the camera motion to the next image. It is important to note that we do not try to learn to reproduce the high gradient points or any standard descriptors based on them, but rather the network learns to find features and descriptors that can be matched well. In fact, the points in the warped image may not even be as detectable based on the gradient as they were in the unwarped image

网络学习的重点在于找到那些**适合匹配的特征点**，而不是普通意义上的那些**hight gradient points**。作者先在一幅图像中提取high gradient points，然后通过warping找到这些点在另一幅图像中的对应点。事实上这些warp到新图像中的点也许并不是该图像上的hight gradient points。

## 网络结构
<div align="center">
<img src="/home/wanghao/blogs/source/all_images/GCN系列/1.png">
</div>

整个网络可以分为两部分：CNN+RCNN。CNN的结构为ResNet-50 + Deconvolution + shortcut的形式结构

### 特征关联
所谓特征关联，其实就是选出正确的特征点，在两张图片中，他们之间的相对位姿已经由ground-truth提供了，所以如果一张图片中的一个特征点能够按照相对位姿投影到另一张图片中对应的特征点上，那么这就是一对正样本，否则就是负样本。具体实现的方式使用的是深度度量学习（**Deep Metric Learning**），核心在于损失函数**triplet loss**的使用
<div align="center">
<img src="/home/wanghao/blogs/source/all_images/GCN系列/2.png">
</div>

其中的f为CNN输出的feature map，![](/home/wanghao/blogs/source/all_images/GCN系列/4.png)是两个feature map的L2距离，m为margin，在实验中被设定为1，x为第一幅图像中的特征点，y\*为正样本，是x点warp到第二幅图像中的点，z\*为负样本，是第二幅图像中除了y以外所有的特征点中与x点的feature map的L2距离最小的点，这个样本的选取策略称为**hardest negative sample mining**。其实就是正样本以外的与anchor距离最近的样本，这个loss的设计有点**类似特征匹配后的radio test**，都是希望最好的匹配与次好的匹配之间存在一定的差距。
<div align="center">
<img src="/home/wanghao/blogs/source/all_images/GCN系列/3.png">
</div>

作者称任何不是正例的match都可以当作是负例，但是选取负例候选集合中最好的作为负例，有助于loss函数，可以加速metric learning.

### Bidirectional RCNN

CNN主要获取图像的空域信息，RCNN主要获取图像的时域信息，将RCNN表达如下
<div align="center">
<img src="/home/wanghao/blogs/source/all_images/GCN系列/5.png">
</div>

作者将关键点的检测看作是一个二分类问题，因此会在网络的顶层计算交叉熵
<div align="center">
<img src="/home/wanghao/blogs/source/all_images/GCN系列/6.png">
</div>

其中o为预测的关键点，cx为2D点的标签(提前在数据集上提取Harris角点得到标签)，实验中![](/home/wanghao/blogs/source/all_images/GCN系列/7.png)和![](/home/wanghao/blogs/source/all_images/GCN系列/8.png)分别为0.1和1.0

## 实验
作者在TUM的fr2数据集上训练，在fr1/fr3上test，数据集中特征点的标签是预先提取Harris角点得到的。下图是作者在TUM和KITTI上对比GCN与其他特征点在motion estimation方面的性能的实验结果
<div align="center">
<img src="/home/wanghao/blogs/source/all_images/GCN系列/9.png">
</div>

<div align="center">
<img src="/home/wanghao/blogs/source/all_images/GCN系列/10.png">
</div>

此外作者实现了一个简单的基于GCN的SLAM系统，并与ORB-SLAM，Elastic Fusion以及RGBDTAM进行了对比
<div align="center">
<img src="/home/wanghao/blogs/source/all_images/GCN系列/11.png">
</div>

# GCNv2
GCNv1存在的问题是**计算开销大以及模型的inference需要多帧图像**。GCNv2相比GCNv1做了以下改进：
* 使用SuperPoint并代替ResNet-50并进行修剪：具体的修改见arxiv的v2版本，这一版本的论文中删除了这一内容

* 去掉RCNN

* 在顶层增加binary activation layer用于生成binary

达到了以下的效果：
* 保证和GCN相近的精度，并大幅减小inference的时间

* 在训练中加入二进制的feature vector来加速匹配，设计和ORB一样的描述子格式

* 在ORB-SLAM2中用GCNv2代替ORB特征并在TX2上达到实时的性能，证明GCNv2的effectiveness and robustness

# 参考
* [论文翻译 GCN-v1:Geometric Correspondence Network for Camera Motion Estimation](https://www.pianshen.com/article/61111891744/)

* [语义SLAM | 深度学习用于特征提取 : GCN-SLAM（一）](https://zhuanlan.zhihu.com/p/68026746)
