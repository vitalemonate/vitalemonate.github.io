---
title: 2017 CVPR sfmlearner
date: 2021-06-14 16:49:54
tags:
categories: depth estimation
---
# 主要贡献
* 提出一个端到端的**基于单目video的**深度估计和运动估计的**自监督学习**框架sfmlearner，是用自监督学习进行深度估计的开山之作，之后的很多算法都是在sfmlearner的基础上进行改进

* 在 KITTI 数据集上的评估结果显示，该方法和之前用 ground-truth pose 或者 depth 进行监督的方法性能是相当的，并且运动估计的结果和现有的通用 SLAM 方法(ORB-SLAM)性能相当

<!--more-->

# 方法
<div align=center>
<img src="/all_images/2017 CVPR sfmlearner/fig0.png">
</div>

上图为sfmlearner的架构，主要由一个depth CNN和一个Pose CNN组成，训练时二者是耦合在一起的，在测试时可以分离使用。sfmlearner的主要思想为，借助view systhesis的方法，将当前帧作为target view，时间上邻近的帧作为source views，通过深度预测模块将source view转移到target view的坐标系下，如下图所示，计算二者的像素误差作为训练的 loss

<div align=center>
<img src="/all_images/2017 CVPR sfmlearner/fig1.png">
</div>

## Explainability mask
上文提出了sfmlearner的loss的构造方法，但是要注意的问题是这个loss的合理性是要在以下3个假设成立的基础上的：

<div align=center>
<img src="/all_images/2017 CVPR sfmlearner/fig5.png">
</div>

如果这三个假设不成立，那么这部分pixel对应的loss就是不合理的，会影响网络的性能，作者在文中提出了一个**explainability mask**来滤除这些不满足假设的pixel所对应的loss，加入explainability mask后的loss的形式为

<div align=center>
<img src="/all_images/2017 CVPR sfmlearner/fig6.png">
</div>

这个mask意义为上述view systhesis的假设成立的概率，由于我们没有对这个mask的监督量，直观上当所有的pixel处的mask都为0时，loss最小，但是这显然不符合我们的期望。

理想情况下， target view和source views中的每个像素点都一一对应，此时mask不起作用，也就是每一个值都是1。作者希望网络尽可能地接近这种情况，也就是说希望mask的每个元素尽可能接近1。为了到达这个目的，作者**用mask和1之间的cross entropy来衡量误差**，希望cross entropy 越小越好

加上平滑loss(depth map二阶导的L1 norm)，最终loss的形式为
<div align=center>
<img src="/all_images/2017 CVPR sfmlearner/fig7.png">
</div>


## 图像warp过程
<div align=center>
<img src="/all_images/2017 CVPR sfmlearner/fig3.png">
</div>

上文提到要将source views warp到target view下，具体的实现参考了[STN](https://arxiv.org/pdf/1506.02025.pdf)的方法，其实就是**inverse image warp + 双线性插值**的方法，关于inverse image warp的内容之前的[笔记](https://github.com/vitalemonate/Image-Warping)里的详细的说明

## Depth CNN
<div align=center>
<img src="/all_images/2017 CVPR sfmlearner/fig2.png">
</div>

上图为sfmlearner的深度估计网络的架构，整体上基于DispNet进行了多尺度的预测，每层的输入通过
<div align=center>
<img src="/all_images/2017 CVPR sfmlearner/fig8.png">
</div>

将网络输出的深度限定为正值并限制在一定范围内

## Pose CNN && Explainability CNN
<div align=center>
<img src="/all_images/2017 CVPR sfmlearner/fig4.png">
</div>

如上图所示，Pose CNN和Explainability CNN共享encoder部分，decoder部分不同，pose CNN的输出为target view与source views之间的相对pose（欧拉角形式的旋转以及3维向量形式的平移）

# 实验部分
## 训练细节
* λs = 0.5/l(l为下采样因子), λs = 0.2

* 除了输出层都使用BN

* 采用Adam优化器，β1 = 0.9, β2 = 0.999，学习率为0.0002，batch size = 4，训练大约在150K次后收敛

* 训练使用的图像大小128×416，测试时可以任意大小

## Depth CNN
* 固定输入序列的长度为3

* 尺度因子：predict的depth map的均值与groundtruth的均值的比值

## KITTI Eigen Split评估depth
<div align=center>
<img src="/all_images/2017 CVPR sfmlearner/fig9.png">
</div>

性能稍差于几个监督学习的方法


## KITTI Odometry评估pose
* 固定输入序列长度为5

* 由于scale ambiguity，在评估时使用最小化ATE的方法优化尺度将轨迹与groundtruth对齐，实验结果如下
<div align=center>
<img src="/all_images/2017 CVPR sfmlearner/fig10.png">
</div>

帧间位姿的估计性能高于ORB-SLAM2

## Explainability mask的作用
<div align=center>
<img src="/all_images/2017 CVPR sfmlearner/fig11.png">
</div>

# 讨论
sfmlearner目前存在的问题：
* 不能显式地解决动态物体和遮挡的问题

* 假设内参已知

* 使用depth map来表示3D场景，未来可以扩展为3D volumeric的表现形式


# 遗留问题
* Depth CNN和DispNet有哪些区别

* smoothness项没有考虑尺度因素

* 训练图像大小的尺寸怎么确定

* 输入序列的长度有什么影响

# 参考
* [arXiv](https://arxiv.org/pdf/1704.07813v2.pdf)

* [Unsupervised Learning of Depth and Ego-Motion](https://zhuanlan.zhihu.com/p/50544334)
