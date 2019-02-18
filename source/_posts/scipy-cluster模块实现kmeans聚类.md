---
title: scipy.cluster模块实现kmeans聚类
date: 2019-02-17 22:03:17
tags:
 - python
 - Kmeans

categories: 基础算法
copyright : true
---

简单验证该模块功能
<!--more-->
***whiten（数组）*** 函数：先将数组各列求标准差，然后将数组每个元素分别除以该标准差

例如：
>[[1,1],
> [1,0],
>[5,5]
>[5,4]]

二维数组，第一列1,1,5,5标准差为2，则该函数输出结果第一列分别为0.5,0.5,2.5,2.5

```
import numpy
from scipy.cluster.vq import *

matrix = [[1, 1], [1, 0], [5, 5], [5, 4]]
numpy_matrix = numpy.array(matrix)
whitened = whiten(numpy_matrix)
print whitened
ret, info = kmeans(whitened, 2)
print ret
print info
classic_info = vq(numpy_matrix, ret)[0]
print classic_info
```

***kmeans(whitened, 2)*** 返回第一个元素为2个中心点，即聚类中心
***vq(numpy_matrix, ret)[0]*** 返回一列数[0,0,1,1]分别表示第一个坐标点属于第0类，第二个属于第0类，第三个属于第一类，第四个属于第一类。