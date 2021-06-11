---
title: IMU预积分的数学理解
date: 2021-06-07 17:58:11
tags:
categories: SLAM
---

# 概述
本文档是学习[On-Manifold Preintegration for Real-Time Visual-Inertial Odometry](https://arxiv.org/pdf/1512.02363.pdf)过程中，自己对IMU预积分以及VIO残差构造的数学理解

<!--more-->

## 状态变量

  定义i时刻的系统状态变量为
  <div align="center">
  <img src="/all_images/IMU预积分的数学理解/fig0.png">
  </div>

## IMU测量模型

  IMU的测量值受加性白噪声以及缓慢变化的bias影响。下图中的前缀为对应的坐标系
  <div align="center">
  <img src="/all_images/IMU预积分的数学理解/fig1.png">
  </div>

## IMU预积分测量模型
  对于关键帧i和j之间的预积分来说

  <div align="center">
  <img src="/all_images/IMU预积分的数学理解/fig5.png">
  </div>

  其中，等式左边为IMU的**预积分测量值**, 右侧为状态量的函数与预积分噪声的广义和(因为涉及到旋转，所以是广义和), 更具体一些，IMU预积分测量模型为

  <div align="center">
  <img src="/all_images/IMU预积分的数学理解/fig6.png">
  </div>

  其中，**预积分测量值完全是由关键帧i,j之间的IMU测量值计算（假设两个关键帧之间的IMU的bias是不变的）得到的**

  <div align="center">
  <img src="/all_images/IMU预积分的数学理解/fig7.png">
  </div>

## 状态估计问题的数学定义

  我们对机器人的状态估计，就是在已知**IMU预积分测量值**的条件下，计算**状态变量**的**条件概率分布**，由贝叶斯法则得
  <div align="center">
  <img src="/all_images/IMU预积分的数学理解/fig3.png">
  </div>

  等式左侧通常称为后验概率，右侧为似然与先验的乘积。使用**后验概率最大化**(Maximize a Posterior, MAP)的方法对状态进行最优估计的结果为
  <div align="center">
  <img src="/all_images/IMU预积分的数学理解/fig4.png">
  </div>

  在先验信息已知且不变的情况下，我们只需要让似然概率最大即可, 我们再写出IMU预积分测量模型

  <div align="center">
  <img src="/all_images/IMU预积分的数学理解/fig5.png">
  </div>

  我们已知IMU预积分的噪声传播方程为
  <div align="center">
  <img src="/all_images/IMU预积分的数学理解/fig8.png">
  </div>

  其对应的协方差矩阵为
  <div align="center">
  <img src="/all_images/IMU预积分的数学理解/fig9.png">
  </div>

  所以观测数据的条件概率为
  <div align="center">
  <img src="/all_images/IMU预积分的数学理解/fig10.png">
  </div>

  它依然是一个高斯分布，为了计算使它最大化的x_i与x_j，我们往往使用最小化负对数的方式，来求一个高斯分布的最大似然。对于一个任意的高维高斯分布![](/all_images/IMU预积分的数学理解/fig11.png)

  其概率密度函数的展开形式为

  <div align="center">
  <img src="/all_images/IMU预积分的数学理解/fig12.png">
  </div>

  取负对数为
  <div align="center">
  <img src="/all_images/IMU预积分的数学理解/fig13.png">
  </div>

  第一项与状态无关，可以略去，于是只要最小化右侧的二次型即可得到对状态的最大似然估计。代入IMU预积分的测量模型得
  <div align="center">
  <img src="/all_images/IMU预积分的数学理解/fig14.png">
  </div>

  该式等价于最小化噪声项的平方（Sigma范数意义下），定义残差项为
  <div align="center">
  <img src="/all_images/IMU预积分的数学理解/fig15.png">
  </div>

  其具体的表达式为
  <div align="center">
  <img src="/all_images/IMU预积分的数学理解/fig16.png">
  </div>

# 参考
* [VIO预积分经典文章](https://arxiv.org/pdf/1512.02363.pdf)

* [视觉SLAM十四讲](https://github.com/gaoxiang12/slambook)
