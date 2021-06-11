---
title: ORB-SLAM3中预积分的实现
date: 2021-06-06 16:16:33
tags:
categories: SLAM
---
# 概要
本文档介绍ORB-SLAM3中预积分的实现过程，主要涉及两个函数：PreintegrateIMU和IntegrateNewMeasurement。调用流程以及对应的功能如下：

<div align="center">
<img src="/all_images/ORB-SLAM3中预积分的实现/fig0.png">
</div>
<!--more-->

# PreintegrateIMU
1. 获取当前帧与上一帧之间的IMU数据，存放在mvImuFromLastFrame
  <div align="center">
  <img src="/all_images/ORB-SLAM3中预积分的实现/fig1.png">
  </div>

    处理后的两个图像帧和IMU的时序关系如下
    <div align="center">
    <img src="/all_images/ORB-SLAM3中预积分的实现/fig2.png">
    </div>

2. 假设两个图像帧之间有n + 1个IMU数据，那么就有n个预积分量，将IMU时间戳与图像时间戳对齐，采用中值积分的方法进行预积分，对IMU取中值分了以下四种情况：

  * 第一个数据但不是最后两帧且imu总帧数大于2
    <div align="center">
    <img src="/all_images/ORB-SLAM3中预积分的实现/fig3.png">
    </div>

  * 中间的正常帧
  <div align="center">
  <img src="/all_images/ORB-SLAM3中预积分的实现/fig4.png">
  </div>

  * 最后一帧且不是第一帧
  <div align="center">
  <img src="/all_images/ORB-SLAM3中预积分的实现/fig5.png">
  </div>

  * 只有两个IMU数据的情况
  <div align="center">
  <img src="/all_images/ORB-SLAM3中预积分的实现/fig6.png">
  </div>

3. 每接收一个IMU数据，调用一次IntegrateNewMeasurement更新预积分量、协方差矩阵以及Jacobian

# IntegrateNewMeasurement
1. 更新dP, dV，未更新dR，因为第(2)步中需要未更新的dR
    <div align="center">
    <img src="/all_images/ORB-SLAM3中预积分的实现/fig8.png">
    </div>

    递推公式的推导为
    <div align="center">
    <img src="/all_images/ORB-SLAM3中预积分的实现/fig9.jpg">
    </div>

2. 根据更新后的dP, dV以及未更新的dR更新预积分噪声的系数矩阵的部分内容
    <div align="center">
    <img src="/all_images/ORB-SLAM3中预积分的实现/fig10.png">
    </div>

    其中预积分噪声![](/all_images/ORB-SLAM3中预积分的实现/fig20.png)是一个9×1的向量，可以由上一时刻的预积分噪声和当前时刻的IMU测量噪声![](/all_images/ORB-SLAM3中预积分的实现/fig21.png)通过线性组合![](/all_images/ORB-SLAM3中预积分的实现/fig11.png)得到，其中A, B矩阵的形式为
    <div align="center">
    <img src="/all_images/ORB-SLAM3中预积分的实现/fig12.png">
    </div>

    在邱笑晨的笔记中推导了预积分噪声是一个多元高斯分布，而IMU测量噪声也是多元高斯分布，二者的线性组合依然是一个多元高斯分布，协方差矩阵为![](/all_images/ORB-SLAM3中预积分的实现/fig18.png)，矩阵的大小为9×9

3. 根据更新后的dP, dV更新对bias的jacobian
    <div align="center">
    <img src="/all_images/ORB-SLAM3中预积分的实现/fig13.png">
    </div>

    对应公式为
    <div align="center">
    <img src="/all_images/ORB-SLAM3中预积分的实现/fig14.png">
    </div>

    递推公式的推导与上文中dP, dV的更新类似，都是将整个求和项分为i~j-2与j-1两部分，具体过程不再赘述

4. 更新dR并更新A，B矩阵中涉及到更新后的dR的部分
    <div align="center">
    <img src="/all_images/ORB-SLAM3中预积分的实现/fig15.png">
    </div>

    dR递推公式的推导为
    <div align="center">
    <img src="/all_images/ORB-SLAM3中预积分的实现/fig16.png">
    </div>

5. 上一步以已获得了A，B矩阵的完整形式，根据![](/all_images/ORB-SLAM3中预积分的实现/fig18.png)更新预积分测量噪声的协方差矩阵。之前的推导都是在各帧之间bias不变的假设下进行的，最终的协方差矩阵考虑了bias的变化————随机游走
    <div align="center">
    <img src="/all_images/ORB-SLAM3中预积分的实现/fig17.png">
    </div>
    其中Nga以及NgaWalk分另为IMU噪声的协方差矩阵以及IMU随机游走的协方差矩阵，由IMU标定值确定，ORB-SLAM3在对外参进行初始化时初始化了这两个矩阵(Cov对应Nga, CovWalk对应NgaWalk)
    <div align="center">
    <img src="/all_images/ORB-SLAM3中预积分的实现/fig30.png">
    </div>

    个人理解：当IMU预积分测量噪声考虑随机游走时，此时噪声的协方差的推导如下（与代码中对应）
    <div align="center">
    <img src="/all_images/ORB-SLAM3中预积分的实现/fig29.jpg">
    </div>

6. 更新dR对陀螺仪bias的jacobian
    <div align="center">
    <img src="/all_images/ORB-SLAM3中预积分的实现/fig22.png">
    </div>

    递推公式的推导为
    <div align="center">
    <img src="/all_images/ORB-SLAM3中预积分的实现/fig23.jpg">
    </div>

    这里有个容易出错的地方是将第一项认为是![](/all_images/ORB-SLAM3中预积分的实现/fig24.png)，但其实![](/all_images/ORB-SLAM3中预积分的实现/fig24.png)的形式为
    <div align="center">
    <img src="/all_images/ORB-SLAM3中预积分的实现/fig25.jpg">
    </div>

    所以要将递归公式继续变形，将![](/all_images/ORB-SLAM3中预积分的实现/fig26.png)以及![](/all_images/ORB-SLAM3中预积分的实现/fig27.png)代入到递推公式中可得
    <div align="center">
    <img src="/all_images/ORB-SLAM3中预积分的实现/fig28.jpg">
    </div>

  至此，ORB-SLAM3中预积分的实现过程就完成了

# 参考
* [On-Manifold Preintegration for Real-Time Visual-Inertial Odometry](https://arxiv.org/pdf/1512.02363.pdf)

* [PetWorm/IMU-Preintegration-Propogation-Doc](https://github.com/PetWorm/IMU-Preintegration-Propogation-Doc)

* [electech6/ORB_SLAM3_detailed_comments](https://github.com/electech6/ORB_SLAM3_detailed_comments)

* [ORB-SLAM3](https://arxiv.org/pdf/2007.11898.pdf)
