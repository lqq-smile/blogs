---
title: K近邻算法详解
date: 2018-06-19 22:46:23
mathjax: true
categories: 人工智能
tags:  
    -K近邻
    -机器学习
---

本文打算简单介绍K近邻算法。不展开论述。

### K近邻算法概述

> 举个例子：
>
> 如果你想知道170厘米身高的人体重是多少。最好的办法是不是从一堆人当中找到几个身高最接近170厘米的人，称出他们的体重然后算平均值呢？

事实上，K近邻就是这样简单。所谓“K”，就是找最接近估算值的K个元素然后算平均值。

设我们有一堆数据：

| 性别 | 身高 | 胖瘦(目测) | 体重 |
| ---- | ---- | ---------- | ---- |
| 男   | 171  | 胖         | 65   |
| 女   | 167  | 瘦         | 59   |
| 男   | 170  | 瘦         | 63   |

现在我们要估测下面一个人的体重：男，169，胖。怎么测呢？我们如果只看身高，女生也有170的人，但明显女生的数据不太符合我们的要求，还有胖瘦也应该符合要预测的数据才行。所以，我们应该用N维的近邻估测。即ML-KNN。

### 数据标准化

这里必须先介绍数据标准化的概念。上面的数据，性别只有男女，即可以改为1，0，胖瘦也是一样，而身高变化的范围却很大。当我们使用算法时某一些字段可能影响过大而造成“不公平”。设$\mu$ 为某字段的均值，$\sigma$ 为对应标准差，数据标准化的公式为：
$$
{ {x- \mu} \over \sigma}
$$
数据标准化后，所有字段都会转化为以0为对称中心的数据，消除了比例差异。

### 欧氏距离

可以使用欧氏距离计算ML-KNN的多维距离。
$$
d= {1 \over n} \sum_{i=1}^n(p_i-q_i)^2
$$

$$
\sqrt{d}
$$

其中,p,q为相应两条数据的对应字段。(博客有问题，无法解析根号)

### N维K近邻
将表格中所有字段进行标准化处理后，再应用欧氏距离公式算出每条样本数据与要估算的数据的值。然后进行排序，找出距离最近的K条数据，然后对这几条数据的体重算平均值。

