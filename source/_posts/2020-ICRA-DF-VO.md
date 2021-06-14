---
title: 2020 ICRA DF-VO
date: 2021-06-14 17:08:59
tags:
categories: depth estimation
---
# 主要贡献
* 回顾了VO的基础，讨论了将深度学习方法与对极几何(2D-2D)以及PnP(3D-2D)进行整合的方法

* 训练了两个CNN分别用来估计单帧图像的深度图以及两帧图像间的光流

* 设计一个简单但是鲁棒的算法DF-VO，在KITTI ODOMETRY上性能超过纯深度学习的方法以及基于几何的方法，并且不会有尺度漂移问题

p.s.:总的来说，这篇文章提出了结合深度学习与传统方法做VO的方法，十分值得学习，但是**作者只提供了用于evaluate的代码，没有提供训练代码**，因此学习暂时只停留在论文的学习层面

<!--more-->

# 基于几何的方法
## 对极几何
在已知2D-2D匹配关系的情况下，可以通过对极几何的方法求解相机的运动
<div align=center>
<img src="/all_images/2020 ICRA DF-VO/fig0.png">
</div>

存在的问题：
* 分解得到的尺度信息是没有尺度信息的

* 无法分解纯旋转的运动

* 如果运动较小，分解得到的解不稳定

## PnP
使用PnP估计帧间运动需要知道3D-2D匹配关系以及3D点的坐标
<div align=center>
<img src="/all_images/2020 ICRA DF-VO/fig1.png">
</div>

# DF-VO
作者提出了一个结合深度学习方法以及上文提到的两种方法的一种VO pipeline DF-VO，算法流程示意图如下
<div align=center>
<img src="/all_images/2020 ICRA DF-VO/fig3.png">
</div>

伪代码如下

<div align=center>
<img src="/all_images/2020 ICRA DF-VO/fig2.png">
</div>

整体流程如下：
* 使用深度估计网络计算图像的深度

* 使用前后两帧的前向光流和反向光流来计算光流一致性
<div align=center>
<img src="/all_images/2020 ICRA DF-VO/fig4.png">
</div>

  以此来确定匹配关系较好的前N=2000对2D-2D匹配点
  <div align=center>
  <img src="/all_images/2020 ICRA DF-VO/fig5.png">
  </div>

  作者在2021年最新提到交arxiv上的[文章](https://arxiv.org/abs/2103.00933)中提到了一个类似于对匹配点对做均匀化的策略，可以增强匹配点对的location diversity以防止欠拟合并且可以防止一些退化情况的出现（匹配点对都在一个平面）

* 此时已经得到了2D-2D的匹配关系以及3D-2D的匹配关系，可以使用上文提到的两种几何方法来计算帧间位姿，但是现在的基于CNN的单目深度估计方法的精度仍然不够好，而基于CNN估计光流的方法很精确并且有着很好泛化能力，这意味着2D-2D的匹配关系要比3D-2D的匹配关系更准确。所以如果光流的大小的均值大于每个阀值（5pixel），那么就使用对极几何的方法计算没有尺度信息的帧间位姿
<div align=center>
<img src="/all_images/2020 ICRA DF-VO/fig6.png">
</div>

  否则就直接使用PnP算法直接估计有尺度信息的帧间位姿（如果从本质矩阵分解得到的位姿不符合**cheirality condition**也要进行PnP），从论文给出的实验结果来看，直接使用PnP的方法得到的结果精度较低

* 恢复尺度信息：如果上一步采用的是对极几何的方法，那么下一步就要三角化出对应的3D点D'，将D'与深度网络估计出的深度做对比可以得到尺度因子s，将这个尺度因子乘在帧间的平移数据上即可得到帧间的带尺度信息的位姿
<div align=center>
<img src="/all_images/2020 ICRA DF-VO/fig6.png">
</div>

  这只是一个简单的恢复尺度的方法，在2021年的[文章](https://arxiv.org/abs/2103.00933)中提到了一个更复杂的方法

## 预测depth的网络
DF-VO中作者采用了同组的[SC-sfmLearner](https://arxiv.org/pdf/1908.10553.pdf)，网络的loss如下
<div align=center>
<img src="/all_images/2020 ICRA DF-VO/fig8.png">
</div>

其中第一项为光度loss，第二项为depth边缘的平滑loss，第三项为尺度一致性loss
* 光度loss
<div align=center>
<img src="/all_images/2020 ICRA DF-VO/fig9.png">
</div>

* depth边缘的平滑loss
<div align=center>
<img src="/all_images/2020 ICRA DF-VO/fig10.png">
</div>

* 逆深度一致性loss
<div align=center>
<img src="/all_images/2020 ICRA DF-VO/fig11.png">
</div>

### 解决尺度一致性的方法：
1.训练时用双目，测试时用单目
2.加入temporal geometry一致性的regularization([SC-sfmlearner](https://arxiv.org/pdf/1908.10553.pdf)以及作者之前的[文章](https://arxiv.org/pdf/1903.00112.pdf)中提出的）

## 预测flow的网络
由于对于每个pair的图片，要计算正向和反向的光流，因此计算光流的网络必须要兼顾速度和精度，DF-VO中作者使用的是轻量级的光流估计网络[LiteFlowNet](https://arxiv.org/pdf/1805.07036.pdf)，这个网络比较精确并且有着不错的泛化能力，训练的loss如下（最新的[文章](https://arxiv.org/abs/2103.00933)loss稍有不同)

<div align=center>
<img src="/all_images/2020 ICRA DF-VO/fig12.png">
</div>

# 实验
## 训练细节
在KITTI Odometry的00-08序列上训练深度网络并finetune光流网络（光流网络是在Scene Flow上训练得到的）
<div align=center>
<img src="/all_images/2020 ICRA DF-VO/fig13.png">
</div>

## 评估
在KITTI上将DF-VO和基于CNN或者传统的方法进行对比，我们使用双目训练，单目测试的结果Stereo Train超过了其他所有方法，结果如下
<div align=center>
<img src="/all_images/2020 ICRA DF-VO/fig14.png">
</div>

几个要点：
* ORB-SLAM2的评估结果是连续跑三遍然后取指标的平均值

* 对于没有尺度因子的方法，通过最小化ATE,进行7DOF（平移+旋转+尺度）的优化得到相应的结果（类似于用evo的评估轨迹的方法）

* 对于有尺度因子的方法，为了公平起见（和没有尺度的方法相比），进行6个自由度的优化

# Ablation Study

果Stereo Train超过了其他所有方法，结果如下
<div align=center>
<img src="/all_images/2020 ICRA DF-VO/fig15.png">
</div>

其中对比的参考模型Reference Model的配置为

<div align=center>
<img src="/all_images/2020 ICRA DF-VO/fig16.png">
</div>

经上表对比可以得到网络的各个模块的作用：
* PnP: 可以看出只用PnP的方法的效果很差，这个要归因于深度网络的精度不够高（今后如果有精度更高的方法出现可以考虑用其他精度更高的网络替换）

* 深度模型：Mono.与参考模型相比使用单目图像去训练，因此得到的平移误差更大，加入尺度一致性的相关loss项(逆深度一致性loss)后可以得到改善，但是二者仍然有scale ambuguity问题

* 光流模型：在KIITI上finetune可以改善性能

* 均匀采样：对预测的光流均匀采样得到2D-2D的匹配结果，这样的效果较差，证明了flow consistency的有效性

* 输入全分辨率的图像：如果输入是全分辨率的图像，预测的光流更精确，因此结果更好


# 参考
* [DF-VO: What Should Be Learnt for Visual Odometry?](https://arxiv.org/abs/2103.00933)
