---
title: 2019 ICCV depth cues
date: 2021-06-14 16:52:19
tags:
categories: depth estimation
---
# 概述

近年来，基于深度学习的单目深度估计算法取得了很大的进展，但是几乎没有人关注网络本身到底从图像上学习到了什么内容。这篇文章通过对一些对深度估计有影响的因素进行对比实验，揭示了基于深度学习的方法进行单目估计的局限性。

作者主要从**物体的位置和大小**、**相机的pose**以及**物体和图像自身的属性**等三个方面进行了对比实验以揭示网络到底是如何从单帧图像中学习得到深度的。用于实验的4个方案分别为：**MonoDepth， SfMLearner，Semodepth以及LKVOLearner**
<!--more-->

# Position vs. apparent size
<div align=center>
<img src="/all_images/2019 ICCV depth cues/fig1.png">
</div>

## 计算物体深度的两种方法

* 给定物体的实际大小**H**和图像中的大小**h**，可以使用以下方法估算距离：

<div align=center>
<img src="/all_images/2019 ICCV depth cues/equation1.png">
</div>

在KITTI数据集中最常遇到的对象来自数量有限的类别（例如汽车，卡车，行人），其中类别内的所有对象的大小大致相同。 因此，网络可能已经学会了识别这些对象并使用它们的大小来估计其距离

* 网络可以使用物体地面接触点的垂直图像位置**y**估算深度。 给定摄像机在地面上方的高度Y，可以通过以下方式估算距离:

<div align=center>
<img src="/all_images/2019 ICCV depth cues/equation2.png">
</div>

该方法不需要任何有关物体真实尺寸**H**的知识，而是假设存在**平坦的地面**和已知的**相机姿态**(Y，yh)。 这些假设也大致适用于KITTI数据集

## 实验设置
作者从KITTI的scene flow数据集中crop出一些对象（主要是车），通过变换大小和位置贴在一张真实的图片中来验证各个猜想

## 实验结果

<div align=center>
<img src="/all_images/2019 ICCV depth cues/fig2.png">
</div>

<div align=center>
<img src="/all_images/2019 ICCV depth cues/fig3.png">
</div>

* Position and size: 除了**LKVOLearner**之外，所有深度估计都表现出预期的效果：估计深度保持接近物体的真实深度，这表明网络仍然可以在这些人造图像上正常工作

* Position only: 当仅改变汽车的垂直位置时，网络仍然可以**粗略估计**到物体的距离，尽管该距离被稍微高估了（MonoDepth， SfMLearner及LKVOLearner）或被低估了（Semodepth）

* Size only: 当仅改变物体的尺寸而使地面接触点保持恒定时，没有网络观察到距离的任何变化。

上述实验结果表明，尽管在删除大小信息时会观察到估计的深度值的一些变化，但是网络主要还是依赖于对象的**垂直位置**，而不是它们的外观大小

# Camera pose: constant or estimated?
这一部分作者探究了camera pose对于深度估计算法的影响。理论上讲，一个鲁棒的算法应该对于camera pose的变化具有不变性。但是实际上并非如此。作者分别对pitch和roll角的扰动进行了分析(因为KITTI中的相机是固定在车身上的，可能发生偏差的是pitch和roll角)

## Camera pitch
### 实验一
这个实验作者意在寻找图像的**真实水平线**（根据Velodyne数据确定）与MonoDepth估计的水平线(通过RANSAC进行拟合得到)之间寻找相关性

<div align=center>
<img src="/all_images/2019 ICCV depth cues/fig4.png">
</div>

结论：虽然预计MonoDepth将完全跟踪地平线或根本不跟踪地平线，但发现回归系数为0.6，这表明它在这些极端之间进行了某些操作（感觉并不能说明什么问题）


### 实验二
由于第一个实验并不能充分说明问题，作者进行了第二个实验，以排除Velodyne数据和真实水平范围较小（±10 px）的潜在问题

作者通过center crop的方式来模拟相机pitch角度的变化
<div align=center>
<img src="/all_images/2019 ICCV depth cues/fig5.png">
</div>

从下图的结果来看，所有网络都能够检测到摄像机俯仰角的变化，但是所有网络都**低估了水平线的变化**
<div align=center>
<img src="/all_images/2019 ICCV depth cues/fig6.png">
</div>

### 实验三
由于网络使用物体的垂直位置来估计深度，因此作者预计这种低估会影响估计的距离。这个实验是为了再次验证这个观点
<div align=center>
<img src="/all_images/2019 ICCV depth cues/fig7.png">
</div>

结果显示估计到的视差确实受pitch变化的影响，结果还表明，网络使用的是物体的垂直图像位置，而不是物体到地平线的距离，因为裁剪图像时后者不会发生变化。
## Camera roll
类似上述对camera pitch的实验，作者通过以下方式对原图像进行crop来模拟roll的变化：
<div align=center>
<img src="/all_images/2019 ICCV depth cues/fig8.png">
</div>

得到的估计的roll shift和真实roll shift偏移的关系如下：
<div align=center>
<img src="/all_images/2019 ICCV depth cues/fig9.png">
</div>

得到的结论与pitch角的变化得到的结论是相似的：所有的网络都能够检测到相机的侧倾角度，但是该角度被低估了，用roll角的变化对结果的影响没有pitch的变化所带来的影响大



# Obstacle recognition
本节主要来探索物体和图像自身的一些属性对深度估计的影响，使用的网络是MonoDepth。之前的实验表明，所有四个网络都使用对象在图像中的**垂直位置**来估计其距离。 估算所需的唯一知识是**物体地面接触点的位置**。 由于不需要其他有关障碍物的知识（例如其现实世界的大小），这表明网络可以估计到任意障碍物的距离。但是事实并非如此，如下图：

<div align=center>
<img src="/all_images/2019 ICCV depth cues/fig10.png">
</div>

对于一些没有在training set中出现过的物体，**网络直接忽略了这些物体**（由此可见网络是真的不可靠啊）

## Color and Texture
下图左边的实验是测试图像的颜色对深度估计的影响，右边的实验是测试纹理对深度估计的影响， 其中：

* Grayscale: 将rgb转为灰度图

* False colors: 将hue和saturation通道用KITTI中的语义rgb通道代替来disturb the colors

* Class-average colors: 用物体的均值代替各个位置的像素值

* Semantic rgb: 语义rgb图

<div align=center>
<img src="/all_images/2019 ICCV depth cues/fig11.png">
</div>

结论：

* 只要图像的HSV空间（Hue, Saturation, Value）中的Value值不发生改变，仅能观察到性能**略有下降**
* 当图像的纹理被移除时，深度估计的性能**大幅下降**

这一部分的实验表明物体的确切颜色无关紧要，而物体内部相邻区域之间的对比度或明暗区域之间的对比度等特征更为重要

## Shape and contrast
### 实验一
<div align=center>
<img src="/all_images/2019 ICCV depth cues/fig12.png">
</div>

结论：

* 物体不需要熟悉的形状或纹理即可被识别

### 实验二
<div align=center>
<img src="/all_images/2019 ICCV depth cues/fig13.png">
</div>

结论：
* 当汽车的内部被移除，只保留边缘时，网络仍然估计出了汽车内部的深度，这表明该网络**主要对物体的轮廓敏感，其余部分则进行填充**
* 除去侧面或底部边缘时，不再检测到汽车。 但是，当仅移除底边时，仍将车辆的侧面检测为两个薄物体

### 实验三
作者怀疑车底部的阴影（在KITTI中非常普遍地存在）为网络主要检测的目标，尝试调整底部阴影区域的brightness和thickness，然后评估深度估计的效果
<div align=center>
<img src="/all_images/2019 ICCV depth cues/fig14.png">
</div>

<div align=center>
<img src="/all_images/2019 ICCV depth cues/fig15.png">
</div>

结论：

* 底部边缘既要厚又要暗才能成功检测
* 完全填充的形状可以提供更好的深度估计

作者在Figure 10中插入的物体下方添加阴影，发现这些物体可以成功地被检测到
<div align=center>
<img src="/all_images/2019 ICCV depth cues/fig16.png">
</div>

# 参考

* [原文](https://openaccess.thecvf.com/content_ICCV_2019/papers/van_Dijk_How_Do_Neural_Networks_See_Depth_in_Single_Images_ICCV_2019_paper.pdf)

* [CNN是靠什么线索学习到深度信息的？——一个经验性探索](https://zhuanlan.zhihu.com/p/95758284)

