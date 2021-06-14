---
title: FlowNet2.0
date: 2021-06-14 16:40:54
tags:
categories: optical flow
---
# 主要贡献
* 发现数据集的训练顺序会影响网络的性能

* 提出了通过stack网络并引入warp操作来提升网络性能的方法，同时可以调整网络的大小实现速度的精度的tradeoff，在时间开销方面可以达到8fps~140fps(Nvidia GTX 1080)

* 提出了一个专门用于处理small motion的子网络，并通过fusion network将其和stacked architecture进行fusion，得到最终的网络架构FlowNet2，这个网络的性能要大幅度高于FlowNet并在Sintel和KITTI的benchmark上与SOTA达到相近的表现

<!--more-->

# 数据集的安排
## FlyingThings3D
FlowNet是在Flying Chairs这个虚拟合成的数据集上进行训练的，这个数据集只有2D的affine transformation，而2015年[DispNet](https://arxiv.org/pdf/1512.02134.pdf)中提出的数据集FlyingThings3D包含了真实的3D运动和光照影响，可以看作是3D版本的Flying Chairs, 示意图如下

<div align=center>
<img src="/all_images/FlowNet2.0/fig0.png">
</div>

## 训练过程中学习率的调整方案

<div align=center>
<img src="/all_images/FlowNet2.0/fig1.png">
</div>

其中S_short就是FlowNetS的训练方法，而S_long以及S_fine是作者新提出的方法，

## Sintel clean数据集下的性能对比

<div align=center>
<img src="/all_images/FlowNet2.0/fig2.png">
</div>

### 结论一：数据集的训练顺序会影响网络的性能
* 尽管FlyingThings3D更realistic，但是只在FlyingThings3D上训练的结果比只在Flying Chairs上训练的结果更差

* 最好的结果是**先在Flying Chairs上进行训练，然后在FlyingThings3D上进行finetune**（如果先在FlyingThings3D上进行训练，在Flying Chairs上进行finetune，结果会怎样呢？）

### 结论二：FlowNetC的性能好于FlowNetS
* 上表中的FlowNetS的指标是从FlowNet原文中得到的，但是经查询后发现FlowNet原文中这个指标应该是4.5而不是表中的4.45，应该是一个小错误

* 与FlowNet原文相比，上表中使用S_short训练的FlowNetC的性能要远好于FlowNetS，这个可能是因此FlowNet原文中两个网络训练的方法不一致引起的，但是这就有一个问题，FlowNet的原文中提到FlowNetC在S_short这种方法下会产生梯度爆炸的问题，因此调整了学习率调整策略，而这里作者并没有提到他是怎么解决这个问题的，直接得到了3.77这个结果

### 改善性能
* 通过调整数据集以及学习率，将FlowNetS的性能提高了25%，将FlowNetC的性能提高了30%。上表中并没有列出同一个网络在同一个训练数据集下，使用不同的学习率调整策略得到的网络的性能，也就是说没有体现出学习率的调整策略对于网络性能提升的影响，个人认为这里性能提升的主要原因应该还是加入了FlyingThings3D数据集

# 堆叠网络
先放一张FlowNet2整体的网络架构图
<div align=center>
<img src="/all_images/FlowNet2.0/fig3.png">
</div>

[这篇文章](https://arxiv.org/abs/1508.05151)发现所有最好的光流预测算法都利用了循环优化的方法。而基于CNN的像素级预测算法结果往往都含有很多噪声和模糊。通常的做法都是利用一些后处理方法对结果进行优化，如FlowNet中，利用传统变分优化方法对FlowNet输出结果进行再优化。那么是否也能够利用CNN来代替后处理方法对结果进行再优化呢？文中对这一问题进行了探究

## 两个网络的堆叠
后一级网络的输入除了输入I_1、I_2外，还有前一级预测的flow，图像I_2经上一级预测的flow的变换图像I_2(w_i)以及误差图像|I_1-I_2(W_i)|，这样可以使新堆叠的FlowNet模块专注去学习I_1与I_2(W_i)之间剩下的运动变换，从而有效的防止堆叠后的网络过拟合，相关的实验结果如下

<div align=center>
<img src="/all_images/FlowNet2.0/fig4.png">
</div>

通过上表的实验结果可以得出以下几个结论：
* warp操作可以改善网络过拟合的问题并提高网络性能

* 在第一个网络后增加一个中间loss会有助于网络训练(前提是Net1参与训练而不是已经是训练好了)

* 最好的结果是第一个网络参数固定不动，只训练第二个网络（包含warp操作）


## 多个不同的网络的堆叠
这里首先想到堆叠多个共享参数的网络，作者在补充材料中证明了通过堆叠多个共享参数的网络并不能显著提高网络的性能

<div align=center>
<img src="/all_images/FlowNet2.0/fig5.png">
</div>

因此，作者采用了使用不同参数的网络进行堆叠，使用FlowNetC作为整个网络的bootstrap模块，即用来接收最初的两张图片，之后堆叠可以调节宽度的FlowNetS，作者的测试了不同宽度的网络的时间及性能，实验结果如下

<div align=center>
<img src="/all_images/FlowNet2.0/fig7.png">
</div>

由上图可知**保留每层3/8的通道数**是一个不错的选择。最快的FlowNet2-s精度与FlowNetS的精度近似，而在Nvidia GTX 1080运算速率可以达到140fps，不同的堆叠方案的性能如下

<div align=center>
<img src="/all_images/FlowNet2.0/fig6.png">
</div>

# 小位移情况下的子网络
如下图所示，FlowNet在真实图片的小位移情况下，结果往往不够理想
<div align=center>
<img src="/all_images/FlowNet2.0/fig8.png">
</div>

因此作者在数据集和网络结构方面都进行了调整

## 新的数据集
提出一个新的数据集ChairsSDHom，这个数据集的图像位移较小，并将此前的堆叠网络FlowNet2-CSS在ChairsSDHom和FlyingThings3D的混合数据上**微调训练**，将结果网络表示为FlowNet2-CSS-ft-sd

## FlowNet2-SD
对FlowNetS进行改造得到名为FlowNet2-SD的网络，专门用来处理小位移的情况
* 将FlowNetS中第一个卷积层的stride由2改为1，并将7x7和5x5的卷积核均换为多层3x3卷积核以增加对小位移的分辨率

* 在反卷积层之间，增加一层卷积层以便对小位移输出更加平滑的光流预测

最后通过一个fusion network将FlowNet2-CSS-ft-sd和FlowNet2-SD的结果进行融合，得到最终的FlowNet2

# 实验
<div align=center>
<img src="/all_images/FlowNet2.0/fig9.png">
</div>

结果显而易见，FlowNet2达到了和SOTA接近的性能，并且可以达到8~140fps的帧速，此外文中还通过将FlowNet2的预测结果直接用于运动分割和动作识别的任务中，证明FlowNet2的精度已完全可以和其他传统算法媲美的程度，已达到可以实际应用的阶段

# 参考
* [FlowNet到FlowNet2.0：基于卷积神经网络的光流预测算法](https://zhuanlan.zhihu.com/p/37736910)

* [FlowNetPytorch](https://github.com/NVIDIA/flownet2-pytorch)
