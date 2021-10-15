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

# 前端跟踪线程
结合论文和代码，将单目配置下前端跟踪的流程图整理如下
<div align="center">
<img src="/all_images/2021 R-AL OV2SLAM/2.png">
</div>

前端跟踪过程中最重要的步骤是实现光流跟踪关键点(图中**kltTracking**)，其中的要点如下：

## 2D keypoints以及3D keypoints的不同处理
* 由于已知3D keypoints对应的3D点在世界坐标系下的坐标，因此先使用匀速运动模型估计当前帧的初始位姿，然后将3D keyoints对应的3D点投影到当前帧，作为该3D keypoints在当前帧的初始位置

* 由于不知道2D keypoints的3D先验信息，所以只能将其在当前帧的初始位置设置为在上一帧中的位置

## 基于网格的特征提取
OV2SLAM将每张图像分为nbwcells*nbhcells个网格，总共的网格的数量为：

<div align="center">
<img src="/all_images/2021 R-AL OV2SLAM/8.png">
</div>

  图中的nmaxdist为网格的大小，以Euroc数据集为例，nmaxdist为35, 图像的分辨率为752×480，图像可划分的**最多网格数为308**

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

* 在图像中其他**没有关键点的网格中**提取特征并计算描述子。作者提供了三种提取特征的方法，分别是GFTT features (Harris corners with Shi-Tomasi method) ，detectGridFAST以及detectSingleScale

### GFTT
这种特征提取的方法并不是严格地要求每个网格内有指定数目的特征点，而是根据需要的特征点的数目以及两个特征点之间最小距离来控制特征点的分布情况，主要步骤如下：

* 设置mask来表示不提取特征的区域：图像的边界 + 以已有特征点的位置为圆心，nmaxdist_为半径的圆，这样保证了特征点之间的间隔，实现类似非极大值抵制的功能，某一帧mask设置的情况如下
  <div align="center">
  <img src="/all_images/2021 R-AL OV2SLAM/9.png">
  </div>

* 接下来只在上图所示的白色区域提取特征，**最多提取特征的数量 = 总网格数 - 当前帧中已经有特征点的网格数**
  <div align="center">
  <img src="/all_images/2021 R-AL OV2SLAM/12.png">
  </div>

  <div align="center">
  <img src="/all_images/2021 R-AL OV2SLAM/10.png">
  </div>

* 对特征点进行亚像素级的优化

* 如果上一步提取到的特征的数量达到最多提取特征的数量的2/3或者最多提取特征的数量少于20就直接返回

* 不满足上一个条件就继续提取特征，此时**最多提取特征的数量 = 总网格数 - 当前帧中已经有特征点的网格数 - 第一步已经提取的特征点的数量**
  <div align="center">
  <img src="/all_images/2021 R-AL OV2SLAM/11.png">
  </div>

* 同时将特征点之间的最小距离调小为原来的一半，此时的mask为
  <div align="center">
  <img src="/all_images/2021 R-AL OV2SLAM/15.png">
  </div>

* 对新提取的特征点进行亚像素级的优化

### detectGridFAST

* 类似GFTT，先设置不需要提取特征的区域mask，发现代码中应该是忘记设置图像边界也不提取特征，修改代码前后的mask对比如下

  原代码：
    <div align="center">
    <img src="/all_images/2021 R-AL OV2SLAM/13.png">
    </div>

  修改后：
    <div align="center">
    <img src="/all_images/2021 R-AL OV2SLAM/14.png">
    </div>

* 在mask后的图像中遍历每个网格提取Fast角点，每个网格中提取到的特征点按响应值从大到小排序，如果最大的响应值大于20，就将该点加入最终的结果

* 根据提取出来的点的数量调整FAST的阀值

* 对特征点进行亚像素级的优化

### detectSingleScale的流程与detectGridFAST类似

# mapping线程
mapping线程主要负责三角化以及局部地图的跟踪，在双目的配置情况下会增加立体匹配的功能。

## 三角化
mapping中的三角化是整个系统中**唯一生成3D点的地方**，包括初始化成功后的那一帧生成**初始地图**，主要功能是用**当前关键帧与共视关键帧**通过三角化产生新的地图点，使得跟踪更稳。

具体过程为：对于当前关键帧中的每一个2D特征点，将当前2D点与**第一次观测到该2D点的关键帧**中的对应2D点进行三角化。相比ORB-SLAM2中将当前关键帧与其**共视程度最高的20帧至少有15个共视点的相邻关键帧**通过词袋进行特征匹配，再将成功匹配的点对进行三角化的策略，二者的效果有待进一步实验对比。下图为ORB-SLAM2的LocalMapping中的三角化搜索范围

<div align="center">
<img src="/all_images/2021 R-AL OV2SLAM/16.png">
</div>

## 局部地图跟踪
局部地图跟踪的主要功能是**增加局部地图中的点在当前帧的观测情况**，其中局部关键帧为**当前关键帧的共视关键帧（一级关键帧），局部地图点为一级关键帧对应的地图点**。而ORB-SLAM2中的局部地图的规模要比OV2SLAM大，ORB-SLAM2的局部关键帧不仅包括了一级关键帧，还包括了每个一级关键帧的共视程度最高的前10个共视关键帧（二级关键帧）以及一级关键帧的父关键帧和子关键帧。更大范围的局部地图可以提高跟踪效果，同时也会带来更大的开销，所以关于局部地图的大小得根据不同的使用场景来调整。

具体匹配的过程是将3D点投影到当前关键帧的像素平面上，与投影点所在的网格内的特征点进行匹配。之后再将地图点进行融合。

# 状态估计线程
状态估计线程的主要功能为优化**局部关键帧的位姿以及3D点的位置**，并滤除冗余的关键帧。

## 局部地图的优化
这里的优化与ORB-SLAM2相近，优化当前关键帧和**与其至少有25个共视点**的相邻关键帧的位姿以及这些关键帧对应的地图点的3D坐标，对于那些不在这些关键帧范围内，但是可以观测到这些地图点的关键帧，也将观测添加到BA中但是不对这些关键帧的位姿进行优化。

在优化的具体实现过程中，作者将地图点的参数化形式由一般的三维坐标改为一个锚点（该地图点在第一次被观测到的关键帧中的像素位置）与逆深度的形式：
<div align="center">
<img src="/all_images/2021 R-AL OV2SLAM/17.png">
</div>

关于采用逆深度的优点可见这篇[博客](https://blog.csdn.net/shyjhyp11/article/details/115376908)。最终代价函数的形式为
<div align="center">
<img src="/all_images/2021 R-AL OV2SLAM/18.png">
</div>

# 闭环检测线程
OV2SLAM采用了在线词袋的方式来进行回环检测，这是一个重要的创新点。主要流程主要分为关键帧预处理、使用iBoW-LCD算法检测候选关键帧、验证候选关键帧、位姿图优化以及looseBA。

## 关键帧预处理以及选取候选关键帧
作者在论文中提到，出于定位的考虑，OV2SLAM并不会跟踪太多的特征点（大概只有200多）。在这里为了更新词袋树，对每幅图像额外提取300个Fast特征并计算其描述子，然后将此关键帧传给iBoW-LCD用于更新词袋树，当关键帧数量大于100帧时，在词袋树中查找当前关键帧的闭环候选关键帧。

## 验证候选关键帧
由于闭环检测中false positive的结果会给系统带来毁灭性的影响，所以必须是尽最大的可能避免这种错误的出现。作者连续采用了3种验证闭环候选帧空间一致性的方法：kNN BF匹配、EM-RANSAC以及P3P-RANSAC，并使用motion-only BA优化当前关键帧的hypothesis pose，如果最终存在30个以上的内点，则候选关键帧验证成功。

## 位姿图优化
目的是将误差均摊到检测到的关键帧与当前关键帧之间的所有关键帧之间。

## looseBA
与ORB-SLAM2中的fullBA相比，OV2SLAM只对受闭环检测影响的关键帧以及地图点进行优化，这样减轻了BA的负担，但是仍然要花费数秒的时间

---
