---
title: 2020 IROS DXSLAM
date: 2021-10-25 10:31:05
tags:
categories: SLAM
---

# 概述
**DXSLAM**全称为 A Robust and Efficient Visual SLAM System with Deep Features，系统的整体架构图如下
<div align="center">
<img src="/all_images/2020 IROS DXSLAM/1.png">
</div>

**DXSLAM**主要在**ORB-SLAM2**中整合了基于CNN的feature，并将其用于**重定位以及闭环检测**。具体的贡献如下：

算法层面：
* 使用CNN(**HF-Net**)提取每张图像的关键点、局部描述子以及全局描述子

* 提出一种基于全局描述子的重定位方法，比传统基于BOW的方法成功率更高且计算量更小

* 训练局部feature的视觉词袋，并结合**局部描述子以及全局描述子**，提出一个高度可靠的闭环检测方法

工程层面：
* 通过使用**OpenVINO**以及**FBoW**(Fast Bag-of-Words)使系统从SIMD指令中获益，可以在**没有GPU以及其他加速器的情况下在CPU上实时运行**，自称是第一个可以在没有GPU支持的情况下达到实时的基于特征法的SLAM系统

效果：
* 实验结果表明本文提出的各个模块性能比baseline高很多，并且整个系统的轨迹误差也更低
<!--more-->

# 具体实现
## 特征提取
特征提取使用的是**HF-Net**，HF-Net的架构如下
<div align="center">
<img src="/all_images/2020 IROS DXSLAM/2.png">
</div>
作者在论文中提到

>  This design choice is not only driven by **functionalities**,but also experimental results showing that the features from **HF-Net are superior** than those from alternative deep CNN feature extractors for the SLAM system.

也就是说使用HF-Net并不仅仅是**受其功能驱动**，而且还因为在SLAM系统中使用HF-Net，**会有比其他CNN更好的表现**。这里提到HF-Net的功能应该指的是同时提取图像特征、局部描述子以及全局描述子的能力，而性能在后文的实验中有对比。

作者使用intel的**OpenVINO**对HF-Net的预训练模型进行优化，在x86 CPU上重新实现了模型的推理，整个模型中只有**双线性插值不能通过OpenVINO优化**，所以将其移到OpenVINO推理之后的**后处理阶段**。

## 训练词袋
在**OpenLORIS-Scene**数据集上使用**增量式**的方法来训练词袋，具体流程如下：
* 首先对每张图像使用HF-Net提取keypoint和对应的local descriptor

* 对每个pair的相邻帧，使用暴力匹配的方法进行匹配，将匹配的点对划分到**已有的同一个叶子结点**中，不匹配的feature**增加一个新的结点**。其中每个结点按照keypoint的得分**最多保留前300个**
* 使用FBoW来构建词袋，训练的词袋是二进制的，可以加快ORB-SLAM2系统初始化时加载词袋的速度(**6s -> 40ms**)

## 使用global descriptor进行重定位
这里涉及到获得图像global descriptor的两种方法：
* 一种是基于BoW的方法，比如ORB-SLAM2在重定位时**聚合local descriptor**，计算词袋向量作为对当前帧的global descriptor

* 另外一种是基于CNN的端到端地计算图像的global descriptor

作者在论文中提到，基于CNN的global descriptor的图像检索**已被广泛证明**，与BoW方法相比，它**对环境和视角变化的鲁棒性要强得多**，所以作者使用HF-Net推理得到的**global descriptor在关键帧数据库中寻找候选帧**，然后使用**PnP+RANSAC**的标准方法在候选帧中筛析匹配点对并计算位姿

## 使用local descriptor以及global descriptor进行闭环检测
## 闭环检测和重定位的区别
由于错误的闭环检测(也就是FP)会给SLAM系统带来毁灭性的影响，所以闭环检测希望precision更高，而后者希望recall更高，Precision和Recall的定义如下
<div align="center">
<img src="/all_images/2020 IROS DXSLAM/3.png">
</div>

其中Precision从预测结果角度出发，描述了预测出来的正例结果中有多少是真实正例，即预测的正例有多少是准确的。Recall从真实结果角度出发，描述了真实正例有多少被挑选了出来。

## DXSLAM的闭环检测的实现
从上边的系统流程图中可以看出，与原始的ORB-SLAM2相比，DXSLAM在闭环的第一步，即寻找闭环候选帧之后加入了**使用global descriptor进一步筛选**的步骤。关于这一步的必要性，作者称

> the BoW matching method aggregates local features by their distributions and **discards their spatial relations**

就是说局部描述子丢弃了局部特征的空间关系，而全局描述符可以保留这些信息，所以这一步是有意义的。关于这一点参考这个[回答](
https://www.zhihu.com/question/31039746/answer/50516028)

# 评估
## 数据集
评估用到了四个数据集：TUM RGB-D、 **OpenLORIS-Scene**以及City Center和New College。TUM用来评估SLAM的**定位精度**，OpenLORIS-Scene这个数据集是2020年这篇论文的作者在ICRA上发表的一个数据集，主要用于评估算法的**鲁棒性**，而后两个数据集用于评估系统**闭环检测的性能**。

## Feature Evaluation
作者首先在OpenLORIS-Scene上评估了HF-Net的性能，对比的对象为手工设计的feature，以及其他基于CNN的feature，并尝试了不同keypoint与descriptor的组合。

**说明：**

* 为了不影响比较feature的性能，在这个评估中**关掉了重定位以及闭环检测**

* 由于OpenLORIS-Scene中的corridor和home序列有较多featureless的场景，纯视觉的SLAM不太可能完全track整个场景，所以主要评估算法其他几个场景的表现(**office + cafe + market**)

* 下图的结果中的**黑点**代表一个**序列**（一个场景下有多个序列）的开始，**蓝点和蓝线**分别代表系统**初始化以及tracking**，每个场景左下角的百分比代表算法的**average correct rate**，值越大代表系统越鲁棒，右下角的浮点数代表**average ATE RMSE**，值越小代表系统精度越高。其中衡量算法鲁棒性的指标定义如下
<div align="center">
<img src="/all_images/2020 IROS DXSLAM/6.png">
</div>


* DS-SLAM是在ORB-SLAM2的基础上移除了动态物体的影响

实验结果如下
<div align="center">
<img src="/all_images/2020 IROS DXSLAM/4.png">
</div>

**结论：**
* **基于CNN的系统的鲁棒性都优于ORB-SLAM2以及DS-SLAM**。这个结论我觉得不太合适，从表中的数据来看，只有**HF-Net+HF-Net以及D2+BRIEF**的组合在office + cafe + market上的鲁棒性（表中的百分数）都超过ORB-SLAM2，其他算法组合的鲁棒性是要低于ORB-SLAM2的

* D2与BRIEF的组合的性能要比原本的D2-Net的性能好，包括鲁棒性和精度

* 与上一条结论不同，HF与BRIEF的组合并不比原本的HF-Net性能好，同样包括包括鲁棒性和精度

* SuperPoint在弱光条件下，比如office序列，提取的特征少导致track失败，但是HF-Net并不会有这种问题

**疑问：**

* 作者在介绍近些看比较出色的feature时提到了SuperPoint、D2-Net以及GCNv2，这里**唯独没有评估GCNv2的性能**，不知道是出于什么原因

* 由于这个数据集是作者自己提出的，然后在这个数据集上验证自己提出的算法比其他算法强，是不是不太有说服力呢？并且只在一个数据集上评估feature的性能，结果可能存在一定的不确定性，如果能够增加在其他数据集上比如EuRoC以及TartanAir的评估，说明力可能会更强

## Re-localization Evaluation
**说明：**

* OpenLORIS-Scene数据集中的office场景有一些**受控的挑战因素**，所以作者在这个场景下评估了重定位的性能

* 评估重定位的指标如下
<div align="center">
<img src="/all_images/2020 IROS DXSLAM/7.png">
</div>

实验结果如下
<div align="center">
<img src="/all_images/2020 IROS DXSLAM/5.png">
</div>

**结论：**

* 在所有情况下DXSLAM的表现都比ORB-SLAM2好

* 只有发生巨大的视角变化时(office-2)，DXSLAM才会重定位失败

## Loop Closure Detection Evaluation
在City Center和New College数据集上评估系统闭环检测的能力，作者增加了一个实验组HF-FBoW，这个算法不使用global descriptor对闭环检测第一步得到的候选帧进行滤除，直接将FBoW得到的**相似度最大**的帧作为候选帧，这个对比实验可以验证**global descriptor对于闭环检测性能的提高**。实验结果如下
<div align="center">
<img src="/all_images/2020 IROS DXSLAM/8.png">
</div>

**结论：**

* HF-Net显著地提高了ORB-SLAM2的回环检测性能

* local descriptor和global descriptor的组合显著地提高了系统的回环检测性能

## Full System Evaluation
### Test on OpenLORIS-Scene
**说明：**

* DXSLAM-no-incre为训练词袋时**不使用增量式**的方法

* 与Feature Evaluation相同，都只关注算法在**office + cafe + market**这几个场景下的表现

评估结果如下
<div align="center">
<img src="/all_images/2020 IROS DXSLAM/9.png">
</div>

**结论：**

* 增量式地训练词袋可以使算法更高效合理(efficient and reasonable)

* DXSLAM在**环境变动情况下**（从一个序列切换到另一个序列）的鲁棒性更好：比ORB-SLAM2跟踪到更多的轨迹并且ATE的RSME更小

### Test on TUM RGB-D
<div align="center">
<img src="/all_images/2020 IROS DXSLAM/10.png">
</div>

**结论：**

* DXSLAM有较强的**抵抗动态物体干扰的能力**，即使没有像DS-SLAM一样将动态物体显式地移除

**疑问：**

* 作者只提供了在TUM的动态场景下的评估结果，**并没有提供一般场景下的评估结果**，结合在Feature Evaluation提出的疑问，DXSLAM在一般场景下的性能还有待进一步评测

## Runtime Performance
**说明：**

* 作者在GPU以及CPU上都进行了评测，使用的GPU为NVIDIA GeForce GTX **1070** GPU (GPU TDP 150 Watts)，CPU为Intel Core **i7-10710U** (CPU TDP 15 Watts, full system TDP 25 Watts)

* 使用的数据集为**OpenLORIS-Scene的office-1序列**，测试时将图像大小调整为**640×480**

评测结果如下
<div align="center">
<img src="/all_images/2020 IROS DXSLAM/11.png">
</div>

**结论：**

* 即使比其他的CNN多了计算global descriptor的步骤，**HF-Net还是更高效**

* 在OpenVINO优化的情况下，HF-Net可以达到46.2ms（**21.6fps**)，相比没有优化的CPU实现减少了68%。但是这只是HF-Net提取特征点和描述子的时间，**不包括SLAM系统track的时间**。ROS版本的代码中特征提取与SLAM的跟踪分别在两个节点，在同样的机器上从摄像头读取图像，最终可以实现以**15fps**的频率输出位姿


# 未来方向
* 通过改进**专用于SLAM**的CNN架构和训练策略来得到更好的feature，比如SEKD

* 整合deep feature到更先进的基于特征点的SLAM，比如ORB-SLAM3

# Related work
## SLAM综述
* [2016 T-RO, Past, present, and future of simultaneous localization and mapping: Toward the robust-perception age](https://ieeexplore.ieee.org/document/7747236)

* [2021 Journal of Global Positioning Systems, A survey of simultaneous localization and mapping with an envision in 6g wireless networks](https://arxiv.org/abs/1909.05214)

## Features
* [2018 CVPR, SuperPoint: Selfsupervised interest point detection and description](https://arxiv.org/abs/1712.07629)

* [2019 CVPR, D2-Net: A trainable CNN for joint detection and description of local features](https://arxiv.org/abs/1905.03561)

* [2019 R-AL, GCNv2: Efficient correspondence prediction for real-time SLAM](https://arxiv.org/abs/1902.11046)

* [2020 arXiv, SEKD: Self-evolving keypoint detection and description](https://arxiv.org/abs/2006.05077)

## Global descriptor
* [2016 CVPR, NetVLAD: CNN architecture for weakly supervised place recognition](https://arxiv.org/abs/1511.07247)

## Features + Global descriptor
* [2019 CVPR, HF-Net: From coarse to fine: Robust hierarchical localization at large scale](https://arxiv.org/abs/1812.03506)

## Dataset
* [2020 ICRA, Are we ready for service robots? the OpenLORIS-Scene datasets for lifelong SLAM](https://arxiv.org/abs/1911.05603)
