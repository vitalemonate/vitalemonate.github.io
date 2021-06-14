---
title: 2016 CVPR DispNet
date: 2021-06-14 16:47:00
tags:
categories: depth estimation
---
# 主要贡献
* 提出三个合成的stereo video数据集，这是第一个large-scale的可以用来训练并评估scene flow方法的数据集

* 提出一个可以实时计算disparity的网络DispNet，这个网络可以达到SOTA的结果

* 将DispNet和FlowNet联合训练，得到第一个用CNN估计scene flow的网络SceneFlowNet（实验结果较差，文中关于SceneFlowNet的架构以及训练过程的介绍都很粗糙）

<!--more-->

# Scene Flow的定义
场景流指空间中场景运动形成的三维运动场，即3D位置坐标+3D运动向量。在已知相机内参和外参的情况下，可以将scene flow投影到相机平面得到optical flow。在camera-independent的情况下，scene flow可以用optical flow, disparity以及disparity change来定义。视差针对的是单个立体帧有效，在图像序列中，像素视差随着时间变化。视差变化数据填补了在仅使用optical flow和static disparity的scene flow的gap
<div align=center>
<img src="/all_images/2016 CVPR DispNet/fig0.png">
</div>

# 三个渲染数据集
使用开源的3D Creation Suite Blender渲染出一系列带有复杂运动模式的物体双目图片。数据集包括双目RGB图、Segmentations、光流图、视差图、视差变化图、运动边界以及相机的内外参数据。这个大的数据集可以分为三个子数据集：FlyingThings3D，Monkaa以及Driving，下图为一个example
<div align=center>
<img src="/all_images/2016 CVPR DispNet/fig1.png">
</div>

# 网络
## 视差估计网络DispNet
网络的总体架构基于FlowNet，但是做了一些调整，下图是DispNet的架构信息(DispNet对应于FlowNetS)
<div align=center>
<img src="/all_images/2016 CVPR DispNet/fig2.png">
</div>

DispNet与FlowNet的主要区别如下：
* DispNet在expanding中过程中增加了额外的卷积层(对应上表中的iconvN)，这些额外的卷积层可以让DispNet预测的视差图比FlowNet预测的视差图更smooth

* DispNet进行了5次上采样（反卷积），最终输出的feature map的分辨率是原图的1/2

## DispNetCorr
基于FlowNetC，作者提出了DispNetCorr，与FlowNetC的主要区别如下：

* DispNetCorr左右两张图像各自经过conv1和conv2，然后再计算水平方向的相关性，而FlowNetC中左右两图各自经过conv1,conv2和conv3，再计算相关性，这样的话DispNetCorr就少一次下采样过程

* 根据对极几何的知识，匹配点只可能出现在极线上，因此DispNetCorr要求输入的图像是已经rectified的双目图像，这样在计算相关性的时候就可以只计算水平方向的相关性，从而降低运算量并能够在计算量允许的情况下计算更大位移处的相关性。作者在文中提到DispNetCorr设定的最大位移为40pixel，由于相关层是原图经过两次下采样得到的，所以对应原图中的最大位移为40*4=160pixel

## 合并FlowNet和DispNet
将预训练的FlowNet和DispNet按照以下方式combine在一起，其中**FlowNet用来预测左右图之间的光流，两个DispNet分别用来预测t和t+1时刻的视差**，在本文提出的三个数据集上进行训练，得到一个可以估计flow，disparity以及额外的disparity change的网络

<div align=center>
<img src="/all_images/2016 CVPR DispNet/fig3.png">
</div>

# 训练
* 选用Adam优化器，取β1 = 0.9, β2=0.999，学习率λ = 1e − 4，在400 000次迭代后每过200 000次迭代就除以2

* 关于多个loss的处理：如果6个loss全都active的话，低层会得到mixed的梯度，因此设计了对loss加权的方案：在开始的时候只有loss6被active，即权重为1，其他loss的权重为0，在训练的过程中逐渐增加高分辨率的权重，deactive低分辨率的权重，这样设计的好处是：“**This enables the network to first learn a coarse representation and then proceed with finer resolutions without losses constraining intermediate features**”

# 实验结果
## DispNet
将DispNet在FlyingThings3D上训练，如果在KITTI 2015上finetune，就加上-K的后缀，实验结果如下
<div align=center>
<img src="/all_images/2016 CVPR DispNet/fig4.png">
</div>

* DispNetCorr1D-K在当时是KITTI-2015的第二好成绩，仅比最好的MC-CNN效果差一点，但是**速度是MC-CNN-acrt的近1000倍，是MC-CNN-fst的接近10倍**

* 尽管DispNetCorr1D-K在KITTI上的性能比较好，但是可以看出在KITTI上finetune后，网络在其他数据集上的性能变差了，这可能是因为KITTI数据集的图像视差较小，最大150pixel，而其他数据集包括很一些较大的视差，比如500以上的视差值，因此**在KITTI上finetune后网络似乎丧失了预测大的位移的能力**

## SceneFlowNet
将SceneFlowNet在作者提出的数据集上进行评估（没有提在哪个数据集上训练），评估结果如下（看着指标挺差）
<div align=center>
<img src="/all_images/2016 CVPR DispNet/fig5.png">
</div>
由于SceneFlowNet在训练过程中要处理很多数据，因此一次forward的速度比较慢，大概需要0.28，是DispNet的5倍，并且还未收敛。作者期待forward速度会随着训练时间而加快

# 参考
* [Scene Flow Datasets数据集： FlyingThings3D, Driving, Monkaa](https://blog.csdn.net/zz2230633069/article/details/92838002)

* [论文阅读笔记之Dispnet](https://blog.csdn.net/kongfy4307/article/details/75212800)

* [深度学习在深度(视差)估计中的应用(2)](https://vincentqin.tech/posts/depth-estimation-using-deeplearning-2/)
