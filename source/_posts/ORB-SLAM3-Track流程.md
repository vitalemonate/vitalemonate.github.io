---
title: ORB-SLAM3 Track流程
date: 2021-06-08 10:10:34
tags:
categories: SLAM
---
# 概述
  本文档总结了ORB-SLAM3中Track的流程，由于整个流程过于复杂，并且逻辑分支较多，因此这里重点分析建图模式下(**mbOnlyTracking=false**)Track的步骤以及各个状态的切换，忽略特征提取、IMU预积分、视觉初始化以及定位模式的相关内容
  <!--more-->

# 流程图
  在建图模式下，Track流程图如下
  <div align="center">
  <img src="/all_images/ORB-SLAM3 Track流程/fig1.png">
  </div>

# 状态机
  在建图模式下，mState的状态切换图如下
  <div align="center">
  <img src="/all_images/ORB-SLAM3 Track流程/fig0.png">
  </div>

# 部分状态转移的情况
## mState从OK切换为R_LOST的情况
  * 纯视觉模式&&上一帧跟踪正常&&当前帧跟踪失败&&地图中关键帧数量大小10

  * IMU模式&&上一帧跟踪正常&&当前帧跟踪失败&&当前帧距离上一次重定位帧超过1秒&&地图中关键帧数量大于10

  * IMU模式&&上一帧跟踪正常&&当前帧跟踪正常&&当前帧TrackLocalMap失败

## mState从R_LOST切换为OK的情况
  * IMU模式&&IMU已经初始化&&当前帧离丢帧未过5秒&&TrackLocalMap成功

  * 纯视觉模式&&重定位成功&&TrackLocalMap成功

## mState从R_LOST切换为R_LOST的情况
  * 纯视觉模式&&重定位成功&&TrackLocalMap失败（一直重复这种状态）

  * IMU模式&&IMU未初始化&&当前帧离丢帧未过5秒

  * IMU模式&&IMU已初始化&&当前帧离丢帧未过5秒&&TrackLocalMap失败

## mState从R_LOST切换为LOST的情况
  * 纯视觉模式&&重定位失败

  * IMU模式&&当前帧离丢帧已过5秒

# 参考
* [ORB-SLAM3 Track函数](https://github.com/UZ-SLAMLab/ORB_SLAM3/blob/master/src/Tracking.cc#L1657)
