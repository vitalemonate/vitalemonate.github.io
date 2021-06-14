---
title: 2019 ICCV monodepth2
date: 2021-06-14 17:00:54
tags:
categories: depth estimation
---
# 概括
本文的网络整体结构与sfm-learner类似，都是由一个depth网络和一个pose网络组成，作者将重点放在loss的设计以及计算方式上，主要做出了以下改进：

* 改进计算光度loss的metric，选择minimum reprojection loss而不是sfm-learner中用到的average reprojection loss，这样可以有效解决因为遮挡导致loss较大的问题

* 在计算多尺度loss时，选择将各个scale的深度图先上采样到图像分辨率再计算loss，减少visual artifacts

* 对于训练过程中遇到的不符合"相机运动，场景静止"假设的像素点，作者提出了一个auto-mask的方法，将这些点从loss的计算中滤除

<!--more-->

# 架构
整体架构和现有的用自监督学习方法进行深度估计的方法相近的架构，由一个depth网络和一个pose网络组成

<div align=center>
<img src="/all_images/2019 ICCV monodepth2/fig0.png">
</div>

# Loss的设计
网络的loss整体上也是和sfm-learner相近，都是由光度loss以及平滑loss两项构成的，monodepth2只是对光度loss进行了部分改进，而对平滑loss
<div align=center>
<img src="/all_images/2019 ICCV monodepth2/fig9.png">
</div>

没有进行改进



## Minimum reprojection loss
<div align=center>
<img src="/all_images/2019 ICCV monodepth2/fig1.png">
</div>

由上图可知，baseline方法在计算多个source images到target image的投影误差时，如果有些点在target image是可见的，但是在部分source images中被遮挡(在I_t-1中被遮挡，在I_t+1中仍然可见)，baseline采用的这种将多个投影误差求均值的方法就会带来较大的loss，于是作者提出了minimum reprojection loss，即在计算多个source images到target的投影误差时，只将最小的误差项加入到loss中，这样就可以很好地解决上述问题。这样得到的光度误差的表达式为

<div align=center>
<img src="/all_images/2019 ICCV monodepth2/fig2.png">
</div>

其中pe()函数为
<div align=center>
<img src="/all_images/2019 ICCV monodepth2/fig12.png">
</div>


下图显示了这个改进的好处

<div align=center>
<img src="/all_images/2019 ICCV monodepth2/fig3.png">
</div>

## 自动滤除静止点
由于光度loss的设计是基于"相机运动，场景静止"的假设的，所以当场景不符合这个假设时，比如相机静止、物体和相机以相同的速度运动或者场景中有动态物体，如下图所示，网络估计的深度的效果就会变差。

<div align=center>
<img src="/all_images/2019 ICCV monodepth2/fig4.png">
</div>

2020 IROS的一篇[文章](https://arxiv.org/pdf/2003.01360.pdf)中的一张图解释了和相机同向运动的物体的深度是无穷远的原因
<div align=center>
<img src="/all_images/2019 ICCV monodepth2/fig5.png">
</div>

作者设计了一种mask将这些与相机相对运动较小的点从loss中滤除

<div align=center>
<img src="/all_images/2019 ICCV monodepth2/fig6.png">
</div>

理解：如果物体与相机的相对运动很小，那么这个物体在连续的两帧中应该出现在图像的同一个位置，所以pe(I_t, I_t')很小接近0，要小于使用估计的帧间位姿以及深度计算出来的重投影误差pe(I_t, I_t'->t)

如果相机静止，那么这种mask会把场景中所有的静态物体都mask掉，mask的效果如下，所以在训练时要先将训练集中的静止帧移除

<div align=center>
<img src="/all_images/2019 ICCV monodepth2/fig7.png">
</div>

## 计算的多尺度loss
现有的方法在decoder的各个低分辨率的尺度上与对应分辨率的图像计算光度loss，作者称这种方法会在**低分辨率图像的较大的弱纹理区域对应的深度图区域出现洞**(没有提供对应的example)，也就是文中提到的texture-copy artifacts。于是作者提出了一个多尺度计算loss的方法，先将低分辨率的深度图上采样到输入图像的的分辨率，再计算光度loss

<div align=center>
<img src="/all_images/2019 ICCV monodepth2/fig8.png">
</div>

将光度loss改进后加上平滑loss，最终的loss为
<div align=center>
<img src="/all_images/2019 ICCV monodepth2/fig10.png">
</div>

其中µ和λ两个loss项对应的权重

# 实验结果
## KITTI
<div align=center>
<img src="/all_images/2019 ICCV monodepth2/fig11.png">
</div>

### KITTI Ablation Study
<div align=center>
<img src="/all_images/2019 ICCV monodepth2/fig13.png">
</div>

# 参考
* [monodepth2 PyTorch代码](https://github.com/nianticlabs/monodepth2)

* [原文](https://arxiv.org/abs/1806.01260)
