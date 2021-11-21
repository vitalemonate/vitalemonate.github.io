---
title: 2018 P-RL Efficient ANMS
date: 2021-11-21 11:06:04
tags:
categories: SLAM
mathjax: true
---
# 概述
本文题目为[**Efficient adaptive non-maximal suppression algorithms for homogeneous spatial keypoint distribution**](https://www.researchgate.net/publication/323388062_Efficient_adaptive_non-maximal_suppression_algorithms_for_homogeneous_spatial_keypoint_distribution)，作者提出了3种高效的自适应非极大值抑制算法：Range Tree ANMS(RT ANMS), K-d Tree ANMS(K-dT ANMS)以及Suppression via Square Covering(SSC)。算法最主要的创新点是提出了一个基于**二分搜索**的初始化搜索范围的确定方法，这种方法可以加速算法的收敛。

下图为本文提出的ANMS与其他非极大值抑制的方法的对比
<div align="center">
<img src="/all_images/2018 P-RL Efficient ANMS/1.png">
</div>

此外作者在文末提到，在BOW，即词袋应用中，**均匀分布的特征点会对整个图像有更好的描述**，这是一个值得注意的点。
<!--more-->

# 改善特征点空间分布的方法
* Bucketing：将图像分为一些固定大小的网格，然后在网格内提取特征。这种方法可以高效地在图像中提取特征，但是不能避免冗余信息的出现

* NMS：抑制在某个半径范围内不是局部极大值的点。但是依然会有点成簇聚集的问题

* ANMS：在迭代过程中**不断进行调整需要抑制区域的大小**。但是现有的ANMS算法的时间复杂度都是平方级别的，不适合像SLAM这种需要实时性能的应用

# 算法实现
## 初始化搜索范围
本文提出的算法在迭代过程中通过**二分搜索**的方式不断调整网格的大小，从而降低了ANMS的时间复杂度。之前的算法SDC也使用二分搜索的方式，但是**正方形网格边长取值的上边界和下边界分别设置为图像的宽度$W_I$以及1**。而本文提出了一种新的确定正方形网格半径（边长的一半）的上界$a_h$和下界$a_l$的方法，加速了算法的收敛速度。

### 确定$a_h$
令$n$为ANMS前特征点数量，$m$为ANMS后特征点的数量。如下图所示，假定在水平方向上，图像被$q$个边长为$2a_h$的网格覆盖，每两个网格中心的距离为$a_h+1$
<div align="center">
<img src="/all_images/2018 P-RL Efficient ANMS/2.png">
</div>

那么图像的宽度$W_I$可以用$a_h$和$q$表达为
<div align="center">
<img src="/all_images/2018 P-RL Efficient ANMS/3.png">
</div>

由$(1)$式可以得到每行的网格数$q$为
<div align="center">
<img src="/all_images/2018 P-RL Efficient ANMS/4.png">
</div>

同理，用每列的网格数$a_h$和$l$表达图像的高$H_I$如下
<div align="center">
<img src="/all_images/2018 P-RL Efficient ANMS/5.png">
</div>

将$l=\frac{m}{q}$代中$(3)$式，并将$q$用$(2)$式替代，可以得到以下关于$a_h$的方程式
<div align="center">
<img src="/all_images/2018 P-RL Efficient ANMS/6.png">
</div>

解这个一元二次方程并保留正解为
<div align="center">
<img src="/all_images/2018 P-RL Efficient ANMS/7.png">
</div>

### 确定$a_l$
网格边长的下界可以通过最坏的点分布来确定。假设所有的特征点分布在$m$个边长为$2a_l$的正方形内，可以得到如下关系
$$m(2a_l)^2=n$$

由上式可以确定$a_l$为
<div align="center">
<img src="/all_images/2018 P-RL Efficient ANMS/8.png">
</div>

## ANMS based on Tree Data Structure
本文提出的三种方法中，前两种使用了额外的数据结构**Tree Data Structure**(TDS) ，包括K-D树以及Rang树。在上一步确定初始的窗口的上界和下界后，即开始进行算法的主要流程
<div align="center">
<img src="/all_images/2018 P-RL Efficient ANMS/9.png">
</div>

主体思路是先将所有的特征点根据响应强度从大到小排序，然后构建成K-D树或者Range树，之后通过二分搜索确定窗口大小，遍历每个没有被排除的特征点，搜索该点在$w$范围内的邻居，并将他们排除。最终当选中的点数达到取值要求时算法结束。

这里有一点需要注意的是，查看代码后发现上述算法中的search range $w$在 K-dT ANMS中设置为二分搜索后**窗口边长的平方**，而在RT ANMS中设置为二分搜索后的**窗口的边长**。前者的实现与论文并不一致，不知道是否是作者在代码实现有误。

## Suppression via square covering
作者提出的第三种方法不需要额外的数据结构，算法的流程如下
<div align="center">
<img src="/all_images/2018 P-RL Efficient ANMS/10.png">
</div>

主体与思路与前两种方法类似，先对特征点排序，然后通过二分搜索确定窗口大小$w$，将图像划分为边长为$\frac{w}{2}$的网格，然后遍历特征点，将特征点划分到网格中，如果网格没有被cover，那就将这个网格以及上下左右四个方向周围的2个网格同时cover。核心代码如下
<div align="center">
<img src="/all_images/2018 P-RL Efficient ANMS/11.png">
</div>

# 实验
## 时间以及空间复杂度
<div align="center">
<img src="/all_images/2018 P-RL Efficient ANMS/12.png">
</div>

## 时间性能
作者从KITTI的Sequence 00中选择1000张图像，对每张图像提取FAST角点，阀值为5，根据响应强度依次选取100~10000个特征点，选取的步长为100，选固定百分比的特征点数量作为输出，百分比从10%到100%，步长为10%，部分实验结果如下
<div align="center">
<img src="/all_images/2018 P-RL Efficient ANMS/13.png">
</div>

从上图的实验结果中可以看出**SSC在速度方面选超其他的方法**(除TopM)。

## 评估初始化策略的性能
<div align="center">
<img src="/all_images/2018 P-RL Efficient ANMS/14.png">
</div>

<div align="center">
<img src="/all_images/2018 P-RL Efficient ANMS/15.png">
</div>

## 评估特征点的Clusteredness
从KITTI中随机先取2000张图像，将图像分为10×10的网格，提取FAST角点(阀值为12)后计算每个网格中点的均值和方差。
<div align="center">
<img src="/all_images/2018 P-RL Efficient ANMS/16.png">
</div>

结论：本文提出的三种方法性能较TopM、Bucketing等方法有明显提升，并且三种方法输出的点的空间分布接近
<div align="center">
<img src="/all_images/2018 P-RL Efficient ANMS/17.png">
</div>

## 在SLAM中的应用
作者使用一个在概念上接近S-PTAM的SLAM系统对比各种算法，使用KITTI benchmark中的指标，测试结果如下
<div align="center">
<img src="/all_images/2018 P-RL Efficient ANMS/18.png">
</div>

结论：
* TopM的性能最差

* Bucketing算法比TopM性能好，但是从来没有比任何一个ANMS的性能强

* ANMS类的算法可以提高系统对运动物体的鲁棒性，比如Seq04（然而并没有看出Seq04的指标与相比其他序列相比有什么区别）

# 参考
* [论文源码](https://github.com/BAILOOL/ANMS-Codes)
