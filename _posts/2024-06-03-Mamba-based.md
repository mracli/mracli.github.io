---
title: Mamba-Frequency based method相关工作阅读
description: 记录从频率角度出发的图像去雨Mamba-based工作
author: momo
categories:
- 图像处理
tags:
- 深度学习
- 图像处理
layout: post
pin: false
math: true
date: 2024-06-03 20:53 +0800
---
## Mamba在图像处理中的应用

Mamba自 2023 年底以来在多个领域衍生多种子方法，同时在图像处理等领域也有一定发展，最近阅读到两篇工作，都从**频率**的角度尝试进行去雨，虽创新程度不高，但也提供Mamba改进的几个视角。

## FreqMamba

Mamba 方法可以实现长距离的建模而相对不容易出现遗忘的情况，但是该方法在原始领域作用为处理**长序列语言建模**，这样简单将 `SSM` 建模方法引入图像处理必然会出现一定割裂，例如其**逐行** `scan` 操作破坏了图像的二维空间属性，这样 `Mamba` 就需要额外处理这种换行引起的空间位置差别过大的关系建模，如下如：

![](assets/img/Mamba-based/VanillaVMamba.png)
_原始mamba直接应用在图像的scan_

FreqMamba 考虑从频率域出发，采用小波包变换，将图像拆分为多级不同频率下的图像，每一级都保留全局的结构信息，因为每个块现在包含所有的图像信息 (丢失一定频率)，可以降低上述逐行扫描过程中不同位置之间的隔阂，具体为：

![](assets/img/Mamba-based/FreqViewScan.png)_scan元素为不同分辨率下的频域图像分量_

具体的，作者给出如下结构：

![](assets/img/Mamba-based/ArchOfProposedMethod.png)_FreqMamba结构_

该方法也采用UNet框架，BasicBlock，为三分支路线，分别为：空间域图像输入SSM，频域图像输入SSM，以及频域下相位谱和频谱卷积提取特征。

## FourierMamba

相比 FreqMamba, 本文作者修改了原始Mamba扫描token的顺序，将其改为之字形扫描 (Zigzag scan)

![](assets/img/Mamba-based/ZigzagScan.png)_将逐行扫描修改为之字形扫描_

作者认为直接使用逐行扫描频域空间，不利于相邻频率之间的特征交互，所以采用如此之字形的方式，使得相邻频率的像素尽可能靠近。

## 两文结果

![](assets/img/Mamba-based/ResultsBetweenMambaBasedMethod.png)_两方法的去雨效果量化结果_

可以看出 MambaBased 方法与现有的 TransformerBased 方法差距仍旧不大，且实际的Mamba训练时长感人，短时间内该方法 (MambaBased) 还需要进行观察。

## 总结

上述方法尝试从频率域的角度实现图像去雨，分别采用直接嵌入频域角度的图像信息和修改Mamba扫描函数来提升图像去雨质量，但是Mamba本身的长距离建模能力能够处理这样的空间隔阂，故两人在不同的角度提出的改进提升效果不明显。使用Mamba这种长距离建模的方法对于去除雨痕是否是关键的仍有待商榷。