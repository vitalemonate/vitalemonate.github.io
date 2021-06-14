---
title: FlowNet1.0
date: 2021-06-14 16:38:10
tags:
categories: optical flow
---
# 主要贡献
* 提出了一个使用CNN端到端预测光流的网络**FlowNet**

* 提出了一个用于训练网络的no-realistic的合成数据集**Flying Chairs dataset**

* 在现有数据集以及合成数据集上进行测试，验证了用深度学习预测光流的可行性

  * 性能比不过SOTA

  * 可以在GPU上实现5-10fps速度，是可以实时估计光流的方法中的SOTA


<!--more-->

# 网络架构

作者在论文中提出了两种结构：FlowNetSimple以及FlowNetCorr
<div align=center>
<img src="/all_images/FlowNet1.0/fig0.png">
</div>

## FlowNetSimple
FlowNetSimple的架构是一个通用的CNN架构，首先将两张图像在通道维度进行concatenate，然后经过一些列的conv、pool、upsample等等又最终refine到和输入图片一样的大小，并得到估计的光流结果，让网络去决定怎么学习图像对之间的运动信息。

原理上讲，只要这个网络足够大，那么就能学习到光流信息，但是不确定像随机梯度下降这样的局部梯度优化可以使网络达到这一点，因此在这种网络的基础上引入两个图像帧之间的相关性计算，得到第二个网络FlowNetCorr

## FlowNetCorr
与FlowNetSimple将输入图像进行concatenate不同，FlowNetCorr先将输入的两张图像同时经过3个卷积层，然后引入一个**correlation layer**, 用于计算两个feature map各个位置处的相似度

## Correlation Layer
对于size都是w×h×c的两个feature map f1, f2，定义f1的x1处与f2处的两个patch的相关性如下：

<div align=center>
<img src="/all_images/FlowNet1.0/fig1.png">
</div>

其中patch的为边长为 **K = 2k + 1**的正方形，计算一次两个patch的相关度需要**c×K^2**次乘法(类似于卷积计算，在通道的维度也要进行点积)，如果遍历计算一个feature map中所有的位置与另一个feature map中所有位置的相关性，共需要**w^2×h^2×c×K^2**次乘法，计算量非常大，作者在文中提出了以下几个减小计算量的操作：

* 对于上文提到的K = 2k + 1的邻域，作者直接令k = 0，即K = 1，相当于两个feature map进行1*1卷积

* 对于f1的每个位置x1，只计算f2中与f1的偏移不超过D = 2d + 1的点的相关度，输出的feature map的维度为**w×h×D^2**，实现过程中取D = 41，在此基础上加入stride的操作，**实际上使用的D = 21**，整个Correlation Layer的操作的示意图如下

<div align=center>
<img src="/all_images/FlowNet1.0/fig2.png">
</div>

代码实现如下(并非官方实现，是github上的一个高star实现，发现有些地方的实现与论文中有所出入，比如论文中并没有conv6_1这个卷积层，没有再深入分析)

<div align=center>
<img src="/all_images/FlowNet1.0/fig3.png">
</div>

<div align=center>
<img src="/all_images/FlowNet1.0/fig4.png">
</div>

*  为了补充图像的信息，将f1通过一个1*1的卷积层后与Correlation Layer生成的feature map在通道方向进行concatenate，得到的feature map送入后续的卷积层

* 注意到论文的架构图中对Correlation Layer进行了sqrt的操作，作者没有在文中提到这里进行sqrt的原因，在代码实现的过程中也没有看到这个操作，看到在flownet2代码的issue中作者提到了这个[问题](https://github.com/lmb-freiburg/flownet2/issues/49#issuecomment-462037012)

## Refinement

<div align=center>
<img src="/all_images/FlowNet1.0/fig5.png">
</div>

这一个模块就是一个类似U-Net的coarse-to-fine的过程，使用反卷积进行上采样并融合之前的feature map以用及上一层predict的feature map，这样就保留了上一级的高级high-level information以及低层的fine local information, 这个过程共重复4次，即上采样了4次，由于之前进行了6次下采样，因此最后一层predict的feature map是input的1/4

相比于计算量更小的**双线性插值**，在这个分辨率下再进行refinement不再能显著能改善结果，因此就将这个分辨率的输出通过双线性插值的方法上采样至图像原来的尺寸作为最终的predict结果

此外，作者在文中还提到了一种refine的方案，这个方案是之前的一篇[文章](http://people.eecs.berkeley.edu/~malik/papers/brox-malik-pami-2010.pdf)的变体，大概意思是说在1/4分辨率的尺寸上使用coarse-to-fine的方法迭代20次上采样到图像的原尺寸，然后在全尺寸上进行5次迭代得到predict的结果。这种方法在上采样的过程中考虑了边缘的平滑度，因此可以得到更平滑的结果以及亚像素级别的估计，但是同时也会增大计算量。实验部分"+v"的后缀表示使用了这种方法进行refine的方案，下图为作者在论文中给出的这种variational refinement的影响，没太看懂这个图想表达的意思，直观地来看，在小运动的情况下FlowNet预测的光流的结果挺差

<div align=center>
<img src="/all_images/FlowNet1.0/fig11.png">
</div>


# 数据集

<div align=center>
<img src="/all_images/FlowNet1.0/fig6.png">
</div>

## 现有数据集
* Middlebury数据集的图像位移非常小，通常低于10pixel

* KITTI数据集虽然包含大位移但是光流groundtruth是稀疏的且数据集是限定的运动下得到的

* Sintel数据集有两个版本：Sintel Final以及Sintel Clean，前者考虑了motion blur以及fog，而后者没有这些影响，此外，Sintel数据集兼顾了大位移以及小位移的情况

获取pixel-wise的真实物体的光流groundtruth数据是很难的，此类数据集也很少，现有的数据集有Middlebury，KITTI以及Sintel，但是用于训练FlowNet的话，由上表的总结可以看出数据量是不够的，因此作者提出了一个合成数据集Flying Chairs，关于数据集的生成方法在论文中有详细的介绍，在此就不再赘述，只放一个Flying Chairs数据集的example

<div align=center>
<img src="/all_images/FlowNet1.0/fig7.png">
</div>

为了方便光流的可视化，论文的附录中介绍了flow field的概念，如下图所示，将光流的方向以及大小分别用色谱中的color以及color intensity进行编码，**每个像素处的光流是从正方形中心到该像素的向量**(对应在网络架构中就是flow5, flow4, flow3的输出feature map的通道数都是2)

<div align=center>
<img src="/all_images/FlowNet1.0/fig8.png">
</div>

# 实验部分
## 训练细节
* 使用Adam为优化方法，β1 = 0.9 and β2 = 0.999，由于每个像素其实都是一个训练样本，因此采用了较小的batch，min-batchsize=8 image pers，初始学习率 λ = 1e−4，在300k次迭代后每过100k次迭代学习率减小一半，在训练FlowNetCorr的过程中发现初始学习率为λ = 1e−4时会发现梯度爆炸，因此选择更小的学习率λ = 1e−6，在10k次迭代后增加到λ = 1e−4，之后按照之前提到的学习率调整策略进行调整

* 训练过程中将最高分辨率处预测结果上采样到groundtruth的图像的尺寸，然后计算EPE(endpoint error)，即预测的光流与ground truth欧氏距离在所有像素上的平均值来得到loss，这里没有使用多尺度的loss计算，不知道如果引入加权的多尺度的loss是否能得到更好的结果

* 网络只在Flying Chairs这个no-realistic的数据集上进行训练，并在Sintel数据集上进行fine tune

# 实验结果
<div align=center>
<img src="/all_images/FlowNet1.0/fig9.png">
</div>

## Sintel数据集
* 在Sintel Clean中，FlowNetC的效果比FlowNetS好，在Sintel Final中，后者的表现更好

* 各版本的训练和测试结果都没有当前的SOTA EpicFlow的效果好

## KITTI数据集
* FlowNetS的效果好于FlowNetC，但是二者的结果依然与SOTA相差较大

* 部分test的结果用"-"表示，作者没有明确说明，但是在论文Comparing the architectures小节的分析中可知，"-"应该表示的是EPE误差较大

## Middlebury数据集
* FlowNetS的效果稍好于FlowNetC，除了FlowNet+ft的对比结果，但是二者的结果依然与SOTA相差较大

## Flying Chairs数据集
* 因为网络就是在这个数据集上训练的，所以test的结果好于SOTA

## 时间因素
* 很多方法只提供了在单cpu上的时间（而我们的网络没法在CPU上跑起来），所以对比了各个版本的网络与提供了GPU时间的方法的时间开销，结果表明FlowNet在GPU上的开销要少于当前实时的光流估计的SOTA EPPM，尤其是**FlowNetS和FlowNet+ft比EPPM快了两倍**

## 结果分析
### 训练数据集
* 作者尝试了只使用Sintel数据集进行训练，发现在test的时候EPE比在Flying Chairs上训练并在Sintel上finetune的网络高了1 pixel

* 尽管Flying Chairs数据集已经很大了，但是数据增强的方法还是有必要的，采用数据增强的方法可以使EPE降低2 pixel

### 不同网络结构的结果对比
* FlowNetS在Sintel Final上的泛化性能强于FlowNetC，FlowNetC在Flying chairs和Sintel Clean上的性能强于FlowNetS。Flying Chairs与Sintel Final相比，没有motion blur以及fog，说明FlowNetC对数据有一点过拟合

* FlowNetC在处理大位移是似乎遇到了些麻烦，FlowNetS+ft对于最大位移超过40pixel的误差达到了43.3px，而FlowNetC+ft对于最大位移超过40pixel的误差达到了48px，这一点是因为网络在设计的时候就设定了max displacement，这一点可以通过增大训练时max displacement值，但是会增大计算开销

<div align=center>
<img src="/all_images/FlowNet1.0/fig10.png">
</div>

# 参考
* [基于FlowNet的光流估计](https://zhuanlan.zhihu.com/p/124400267)

* [FlowNetPytorch](https://github.com/ClementPinard/FlowNetPytorch)

* [Pytorch-Correlation-extension](https://github.com/ClementPinard/Pytorch-Correlation-extension)
