---
title: 2018 P-RL Efficient ANMS
date: 2021-11-21 11:06:04
tags:
categories: SLAM
mathjax: true
---
# 概述
本文题目为[**Efficient adaptive non-maximal suppression algorithms for homogeneous spatial keypoint distribution**](https://www.researchgate.net/publication/323388062_Efficient_adaptive_non-maximal_suppression_algorithms_for_homogeneous_spatial_keypoint_distribution)，作者提出了3种高效的自适应非极大值抑制算法：Range Tree ANMS(RT ANMS), K-d Tree ANMS(K-dT ANMS)以及Suppression via Square Covering(SSC)。算法主要有两点contribution:
* 提出一个方形的搜索范围来减小ANMS的计算复杂度

* 提出一种新的基于二分搜索的初始化搜索范围的确定方法，这种方法可以加速算法的收敛

下图为本文提出的ANMS与其他非极大值抑制的方法的对比
<div align="center">
<img src="/all_images/2018 P-RL Efficient ANMS/1.png">
</div>

<!--more-->
