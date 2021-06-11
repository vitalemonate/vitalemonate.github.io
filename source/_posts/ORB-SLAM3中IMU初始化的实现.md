---
title: ORB-SLAM3中IMU初始化的实现
date: 2021-06-06 16:31:27
tags:
categories: SLAM
---
# LocalMapping流程概述
<div align="center">
<img src="/all_images/ORB-SLAM3中IMU初始化的实现/fig0.png">
</div>

<!--more-->
# InitializeIMU
1. 对状态变量![](/all_images/ORB-SLAM3中IMU初始化的实现/fig1.png)进行初始化

<div align="center">
<img src="/all_images/ORB-SLAM3中IMU初始化的实现/fig2.png">
</div>

* 尺度因子：初值为1.0

* bias：第一次InitializeIMU时初值为0，之后使用关键帧对应的bias

* 视觉世界坐标系下的重力方向dirG：各个关键帧的速度变化量的平均方向，而R_wg描述的是真正的世界坐标系（东北天坐标系）下的重力方向(0, 0, -G=-9.81)到视觉世界坐标系下的重力方向dirG的旋转

* 各个时刻的速度：用当前关键帧与上一关键帧之间IMU的位置变化量（视觉世界坐标系下的）除以两帧之间的时间

2. 进行Inertial-only MAP Estimation优化
  <div align="center">
  <img src="/all_images/ORB-SLAM3中IMU初始化的实现/fig3.png">
  </div>

  * 对应的因子图为
  <div align="center">
  <img src="/all_images/ORB-SLAM3中IMU初始化的实现/fig20.png">
  </div>

  * IMU误差项
  <div align="center">
  <img src="/all_images/ORB-SLAM3中IMU初始化的实现/fig22.png">
  </div>

  * 关于先验的处理
  <div align="center">
  <img src="/all_images/ORB-SLAM3中IMU初始化的实现/fig21.png">
  </div>

  * 先验信息的残差计算
  <div align="center">
  <img src="/all_images/ORB-SLAM3中IMU初始化的实现/fig23.png">
  </div>
  <div align="center">
  <img src="/all_images/ORB-SLAM3中IMU初始化的实现/fig24.png">
  </div>


3. 应用优化后的变量

  <div align="center">
  <img src="/all_images/ORB-SLAM3中IMU初始化的实现/fig4.png">
  </div>


  * ApplyScaledRotation
    <div align="center">
    <img src="/all_images/ORB-SLAM3中IMU初始化的实现/fig5.png">
    </div>

    用优化后的R_wg将视觉世界坐标系的z轴与东北天坐标系的z轴对齐，即对齐两个坐标系的重力方向，这部分推导过程如下：
    <div align="center">
    <img src="/all_images/ORB-SLAM3中IMU初始化的实现/fig6.png">
    </div>

    将各个关键帧的pose，速度以及地图点坐标转换到新的坐标系下，并用优化后的尺度因子更新这些变量

    * 对于每个关键帧：
    <div align="center">
    <img src="/all_images/ORB-SLAM3中IMU初始化的实现/fig7.png">
    </div>

    * 对于每个关键帧的速度以及地图点（省略齐次与非齐次坐标间的转换）：
    <div align="center">
    <img src="/all_images/ORB-SLAM3中IMU初始化的实现/fig8.png">
    </div>

  * UpdateFrameIMU

    * 应用更新后的bias
    <div align="center">
    <img src="/all_images/ORB-SLAM3中IMU初始化的实现/fig9.png">
    </div>

    * 用预积分量更新当前关键帧以及上一帧的IMU的绝对状态量（世界坐标系下的旋转，位置以及速度）
    <div align="center">
    <img src="/all_images/ORB-SLAM3中IMU初始化的实现/fig10.png">
    </div>
    
      这部分的推导可由预积分的定义式得到
      <div align="center">
      <img src="/all_images/ORB-SLAM3中IMU初始化的实现/fig11.png">
      </div>

4. 进行FullInertialBA（对应IMU初始化的第三步Visual-Inertial MAP Estimation）并更新相关状态量

  <div align="center">
  <img src="/all_images/ORB-SLAM3中IMU初始化的实现/fig12.png">
  </div>

  这里的FullInertialBA优化关键帧位姿，速度，bias以及地图点的坐标，如果priorA不为0，假设各帧之间的IMU的bias相同，因此每个IMU误差项都有6个相关变量（上一帧以及当前帧的位姿，速度，还有两帧公用的bias)，此外还要单独考虑先验信息，此时对应的因子图为
  <div align="center">
  <img src="/all_images/ORB-SLAM3中IMU初始化的实现/fig15.png">
  </div>

  * IMU误差项
  <div align="center">
  <img src="/all_images/ORB-SLAM3中IMU初始化的实现/fig13.png">
  </div>

  * 先验信息
  <div align="center">
  <img src="/all_images/ORB-SLAM3中IMU初始化的实现/fig14.png">
  </div>

  * 视觉误差项
  <div align="center">
  <img src="/all_images/ORB-SLAM3中IMU初始化的实现/fig16.png">
  </div>

# ScaleRefinement
  从IMU初始化第25秒开始到75秒内，如果关键帧的个数少于100，那么每隔10秒进行一次ScaleRefinement，这个优化和Inertial-only MAP Estimation类似，只是优化变量只有尺度因子以及重力方向R_wg
  <div align="center">
  <img src="/all_images/ORB-SLAM3中IMU初始化的实现/fig17.png">
  </div>

  对应的因子图为
  <div align="center">
  <img src="/all_images/ORB-SLAM3中IMU初始化的实现/fig18.png">
  </div>

  对照代码发现在进行ScaleRefinement时使用的bias都是第一个关键帧的bias，与因子图的形式有差异
  <div align="center">
  <img src="/all_images/ORB-SLAM3中IMU初始化的实现/fig19.png">
  </div>
