# ﻿Redis高级数据类型-Bitmap和HyperLogLog


## 位图

位图不是特殊的数据结构，它的内容其实就是普通的字符串，也就是 byte 数组。我们可以使用普通的 get/set 直接获取和设置整个位图的内容，也可以使用位图操作 getbit/setbit 等将 byte 数组看成「位数组」来处理。

![img](/images/Redis-advanced%20data%20types/1645926f4520d0ce)

当我们要统计月活的时候，因为需要去重，需要使用 set 来记录所有活跃用户的 id，这非常浪费内存。这时就可以考虑使用位图来标记用户的活跃状态。每个用户会都在这个位图的一个确定位置上，0 表示不活跃，1 表示活跃。然后到月底遍历一次位图就可以得到月度活跃用户数。不过这个方法也是有条件的，那就是 userid 是整数连续的，并且活跃占比较高，否则可能得不偿失。

## HyperLogLog 

`HyperLogLog`，下面简称为`HLL`，它是 `LogLog` 算法的升级版，作用是能够提供不精确的去重计数。存在以下的特点：

- 代码实现较难。
- 能够使用极少的内存来统计巨量的数据，在 `Redis` 中实现的 `HyperLogLog`，只需要`12K`内存就能统计`2^64`个数据。
- 计数存在一定的误差，误差率整体较低。标准误差为 0.81% 。
- 误差可以被设置`辅助计算因子`进行降低。

### 为什么用HyperLogLog

如果要实现这么一个功能：

> 统计 APP或网页 的一个页面，每天有多少用户点击进入的次数。同一个用户的反复点击进入记为 1 次。

用 `HashMap` 这种数据结构就可以，假设 APP 中日活用户达到`百万`或`千万以上级别`的话，我们采用 `HashMap` 的做法，就会导致程序中占用大量的内存。

估算下 `HashMap` 的在应对上述问题时候的内存占用。假设定义`HashMap` 中 `Key` 为 `string` 类型，`value` 为 `bool`。`key` 对应用户的`Id`,`value`是`是否点击进入`。明显地，当百万不同用户访问的时候。此`HashMap` 的内存占用空间为：`100万 * (string + bool)`。

### HyperLogLog原理

如图，给定一系列的随机整数，我们记录下低位连续零位的最大长度 k，通过这个 k 值可以估算出随机数的数量。

![image-20200716222413792](/images/Redis-advanced%20data%20types/image-20200716222413792.png)

HyperLogLog与伯努利试验有关，具体可参考[HyperLogLog 算法的原理讲解](https://juejin.im/post/5c7900bf518825407c7eafd0)

## 参考

[HyperLogLog 算法的原理讲解](https://juejin.im/post/5c7900bf518825407c7eafd0)

[Redis 深度历险：核心原理与应用实践](https://juejin.im/book/5afc2e5f6fb9a07a9b362527)


