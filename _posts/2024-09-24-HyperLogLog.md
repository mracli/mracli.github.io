---
title: HyperLogLog
description: HyperLogLog，概率模型拟合列表存储的基数，物美价廉
author: momo
categories:
- 算法
tags:
- 算法
- Redis
layout: post
pin: false
math: true
date: 2024-09-24 15:19 +0800
---
## 介绍

HLL, HyperLogLog 是一种基于概率统计的基数计算模型，例如可用于高效统计一个网站单日访问用户的数量，或用于统计每个单独的key出现次数(Redis中常用). 最简单的理解可认为是一种抛硬币问题，连续出现1的几率会逐渐降低，但随着抛硬币次数增加，最后会收敛于 $2^{-n}$, 借此可以估计 `n` 的大小，也就是抛硬币次数 ；同理，统计访客(键)对应的哈希值，随着访客次数增加，哈希值低位连续出现1的频率也会服从于总不重复访客数量(n)对应的概率 $2^{-n}$. 

相比直接使用set存储对应的键，然后统计内部元素数量得到该集合的基，使用HLL可以最大程度降低存储开销，如HLL名称所示，其存储开销可认为是样本总体基为 `N`, 那么HLL的存储开销为 `O(log(log(N)))`

## 常见的实现
1. 统计哈希值最高位的1下标(从大到小，从1逐渐增大)，根据下标来判断当前估计的集合基数；
2. 统计哈希值最低位连续0的个数；

两种方式统计随添加的元素数量增加，趋于等价的结果。

## 优化
### 概率模型极小事件导致误差
虽然说HLL用小空间可以相对较为精准的估计出大量数据下的集合基数，但在小数据规模则容易出现较大的误差。
#### 解决方法
统计的哈希值放入多个bucket中，并计算每个bucket的调和平均数，这样可以降低因为小概率事件导致误差较大的问题。
#### 具体实现
哈希值分为两部分，其中一部分用于确认放入哪个bucket，另外一部分用于统计连续0的个数。

![](assets/img/HyperLogLog/HLL_with_bucket_impl.png)

### 小概率事件减少存储
前面说到统计哈希值低位连续0的个数或最高位1的下标，但连续多个数字的事件是小概率事件，实际上存储的bucket可以分为存储低位的连续存储(DenseBucket)，和存储高位的哈希表存储(OverflowBucket)，可以大幅降低设置多个bucket导致存储开销明显增加。

这里可以固定两种 bucket 存储的位数，例如统计出的连续数字个数为 33, 当前存储低位部分的bucket只负责低4位，而存储高位的部分只负责高3位，并通过哈希值确认存放的bucket 下标为 3,
那么 `DenseBucket[3] = 1, OverflowBucket[3] = 2`. 

高位存储采用哈希表的稀疏存储方式，不会因为设置过多的DenseBucket存储位数的影响而占用过多存储。

## 变种
HyperLogLog++, 相比HLL区别在：
- 计算的hash值从32位增加到64位，可提高统计基数的最大上限
- 计算公式中添加的偏差修正系数
- 稀疏的存储方式进一步降低存储(就是上面所说的优化)

## 参考
1. [HyperLogLog](https://engineering.fb.com/2018/12/13/data-infrastructure/hyperloglog/)
2. [Video](https://www.youtube.com/watch?v=lJYufx0bfpw)
3. [维基百科](https://en.wikipedia.org/wiki/HyperLogLog)