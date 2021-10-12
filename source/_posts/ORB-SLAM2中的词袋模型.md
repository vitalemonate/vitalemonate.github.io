---
title: ORB-SLAM2中的词袋模型
date: 2021-10-11 21:36:42
tags:
categories: 其他
---
#  概述
学习和开发ORB-SLAM2两年以来还未深入学习过词袋模型的实现，恰好最近在学习OV2SLAM的在线词袋，就回头对比学习一下ORB-SLAM2中词袋的实现与应用，本文的内容主要是从[小六的注释版ORB-SLAM2的讲义](https://github.com/electech6/ORB_SLAM2_detailed_comments/blob/master/%E3%80%8AORB-SLAM2%E6%BA%90%E7%A0%81%E8%A7%A3%E6%9E%90%E3%80%8B%E5%AD%A6%E4%B9%A0%E6%89%8B%E5%86%8C.pdf)中搬运过来的。
<!--more-->

# 计算词袋向量mBowVec以及mFeatVec
ORB-SLAM2的词袋模型中最重要的两个变量就是mBowVec以及mFeatVec，二者是Frame以及KeyFrame的类成员：

<div align="center">
<img src="/all_images/ORB-SLAM2中的词袋模型/1.png">
</div>

在前端跟踪中的Relocalization、TrackReferenceKeyFrame以及**创建关键帧**时都会被调用。

## mBowVec
mBowVec其实是一个 **std::map<WordId, WordValue>** 。

WordId 和 WordValue 表示Word在所有叶子中距离最近的叶子的id 和权重，其中WordValue是累加更新的：

<div align="center">
<img src="/all_images/ORB-SLAM2中的词袋模型/2.png">
</div>

mBowVec主要用在重定位以及回环检测中:

* 作者用**反向索引**记录每个叶节点对应的**图像编号**。当识别图像时，根据反向索引选出有着公共叶节点的备选图像并计算得分，而不需要计算与所有图像的得分
<div align="center">
<img src="/all_images/ORB-SLAM2中的词袋模型/6.png">
</div>

* 计算两个图像的相似度，比如在计算当前帧与候选关键帧的相似度

<div align="center">
<img src="/all_images/ORB-SLAM2中的词袋模型/5.png">
</div>


## mFeatVec
mFeatVec其实是一个 **std::map<NodeId, std::vector<unsigned int> >** 。

其中NodeId 并不是该叶子节点直接的父节点id，而是**距离叶子节点深度为level up对应的node 的id**。为什么不直接设置为父节点？因为后面搜索该Word 的匹配点的时候是在和它具有同样node id下面所有子节点中的Word 进行匹配。所以搜索范围大小是根据level up来确定的，level up 值越大，搜索范围越广，速度越慢；level up 值越小，搜索范围越小，速度越快，但能够匹配的特征就越少。在ORB-SLAM2中，设定level的大小为4，即Node Id为word向上找4级的node的id。

另外 std::vector 中实际存的是NodeId下所有特征点**在图像中的索引**：

<div align="center">
<img src="/all_images/ORB-SLAM2中的词袋模型/3.png">
</div>

FeatureVector主要用于不同图像特征点快速匹配，加速几何关系验证，比如ORBmatcher::SearchByBoW 中只对NodeId相同的特征点进行特征匹配，避免了暴力匹配，大大加快了匹配的速度，这种加速特征匹配的方法在ORB-SLAM2中被大量使用。

<div align="center">
<img src="/all_images/ORB-SLAM2中的词袋模型/4.png">
</div>

# 参考
* [小六的ORB-SLAM2注释版代码以及讲义](https://github.com/electech6/ORB_SLAM2_detailed_comments)

* [【ORB-SLAM2代码】（五）DBoW2词袋模型](https://blog.csdn.net/hzwwpgmwy/article/details/83477990)

---
