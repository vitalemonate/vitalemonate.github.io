---
title: 2021 R-AL OV2SLAM
date: 2021-10-12 10:17:47
tags:
categories: SLAM
---
#  概述
OV2SLAM的全称为 A Fully **Online and Versatile Visual** SLAM for Real-Time Applications，整体的架构图如下
<div align="center">
<img src="/all_images/2021 R-AL OV2SLAM/1.png">
</div>

OV2SLAM是一个完备的视觉SLAM框架，整合了当新最新的视觉定位成果，可以处理单目、双目，不同尺度的以及不同帧率（从几Hz到几百Hz)的场景。在3个公开数据集上与一些知名的SLAM算法相比，实现了SOTA的精度与实时性能。

<!--more-->
# 特点
* 实时SLAM：可以支持从20Hz~200Hz，并且在后文的实验中可以看出，相比于ORB-SLAM2，OV2SLAM可以在高帧率的情况下依然保持不错的精度；

* 前端采用了LK光流跟踪的方法进行track，这是OV2SLAM中比较重要的部分，主要有以下要点：

  * OV2SLAM并不像传统的基于特征的SLAM对每帧图像都提取指定数量的特征点，普通帧只跟踪上一帧中被跟踪到的特征点。跟踪成功后如果决定当前帧是关键帧，则进一步对当前帧中没有特征点的区域提取特征用于之后的跟踪

  * 特征点分为**2D keypoints以及3D keypoints**，2D特征点是指没有三角化过的特征点，没有其3D位置的先验信息，而3D keypoints是指已经三角化过的点，具有3D点的先验信息。这二者在光流跟踪的过程中会有不同的处理，具体细节见下文

* 闭环检测使用在线词袋iBoW-LCD，在建图跟踪的过程中增量式地更新构建词袋，应该是第一个使用在线词袋的SLAM算法

# 前端跟踪
结合论文和代码，将单目配置下前端跟踪的流程图整理如下
<div align="center">
<img src="/all_images/2021 R-AL OV2SLAM/2.png">
</div>

前端跟踪过程中最重要的步骤是实现光流跟踪关键点(图中**kltTracking**)，其中的要点如下：

## 2D keypoints以及3D keypoints的不同处理
* 由于已知3D keypoints对应的3D点在世界坐标系下的坐标，因此先使用匀速运动模型估计当前帧的初始位姿，然后将3D keyoints对应的3D点投影到当前帧，作为该3D keypoints在当前帧的初始位置

* 由于不知道2D keypoints的3D先验信息，所以只能将其在当前帧的初始位置设置为在上一帧中的位置

## 基于网格的特征提取
OV2SLAM将每张图像分为nbwcells*nbhcells个网格，对于每个网格，取Shi-Tomasi得分最高的点作为特征点。总共的网格的数量为：

<div align="center">
<img src="/all_images/2021 R-AL OV2SLAM/8.png">
</div>

  图中的nmaxdist为网格的大小，以Euroc数据集为例，nmaxdist为35, 图像的分辨率为752×480，图像可划分的**最多网格数为308**，也就是单帧图像最多有308个特征点，数量要远小于ORB-SLAM2(每帧提取1000个特征点)。

## 类Frame的设计
OV2SLAM在整个跟踪过程中并没有像ORB-SLAM2那样每一帧都构造一个Frame类来存放特征点、描述子以及位姿等信息，只保留了一个指向当前帧的指针**pcurframe**，用于实时记录当前帧跟踪到的2D、3D特征点以及位姿信息。接受到新图像时只更新时间戳和帧号，在光流跟踪以及计算位姿后分别更新当前帧的特征点以及位姿信息。

更新时间戳和帧号
<div align="center">
<img src="/all_images/2021 R-AL OV2SLAM/3.png">
</div>

更新特征点的位置信息以及在网格中的分布情况
<div align="center">
<img src="/all_images/2021 R-AL OV2SLAM/4.png">
</div>

<div align="center">
<img src="/all_images/2021 R-AL OV2SLAM/6.png">
</div>

更新位姿
<div align="center">
<img src="/all_images/2021 R-AL OV2SLAM/5.png">
</div>

## 创建关键帧
当一个帧从普通帧变为关键帧时，关于特征点的处理主要有以下两点：

* 对于已经跟踪到的2D、3D特征点，在当前图像中**计算关键点的描述子**，其实就是更新了描述子
<div align="center">
<img src="/all_images/2021 R-AL OV2SLAM/7.png">
</div>

* 在图像中其他**没有关键点的网格中**提取特征并计算描述子，额外提取的特征点数量 = 设定的最大特征数 - 已经有特征点的网格数，其中最大特征数也就是图像所可以划分的最大风格数。

The Mapping works at the keyframes' rate and ensures continuous localization by populating the sparse 3D map and minimize drift through a local map tracking step. Bundle Adjustment is applied with an anchored inverse depth formulation, reducing the parametrization of 3D map points to 1 parameter instead of 3. Loop Closing is performed through an Online Bag of Words method thanks to iBoW-LCD.



---
