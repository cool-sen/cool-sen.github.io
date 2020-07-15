# ﻿Redis数据类型-跳跃表


跳表（skiplist）是一个特殊的链表，相比一般的链表，有更高的查找效率，其效率可比拟于二叉查找树。

## 跳跃表来源

跳跃表在 1990 年由 William Pugh 提出，而红黑树早在 1972 年由鲁道夫·贝尔发明了。红黑树在空间和时间效率上略胜跳表一筹，但跳跃表实现相对简单得到程序猿们的青睐。Redis 和 Leveldb 中都有采用跳跃表。

以下是个典型的跳跃表例子

![image-20200715223005411](/images/Redis-skiplist/image-20200715223005411.png) 

按照上面生成链表的方式，上面每一层链表的节点个数，是下面一层的节点个数的一半，这样查找过程就非常类似于一个二分查找，使得查找的时间复杂度可以降低到O(log n)。

但是，这种方法在插入数据的时候有很大的问题。新插入一个节点之后，就会打乱上下相邻两层链表上节点个数严格的2:1的对应关系。如果要维持这种对应关系，就必须把新插入的节点后面的所有节点（也包括新插入的节点）重新进行调整，这会让时间复杂度重新蜕化成O(n)。删除数据也有同样的问题。

下面看Redis 跳跃表的实现，如何解决的这个问题。

## Redis 跳跃表的实现

为了满足自身的功能需要， Redis 基于 William Pugh 论文中描述的跳跃表进行了以下修改：

1. 允许重复的 `score` 值：多个不同的 `member` 的 `score` 值可以相同。
2. 进行对比操作时，不仅要检查 `score` 值，还要检查 `member` ：当 `score` 值可以重复时，单靠 `score` 值无法判断一个元素的身份，所以需要连 `member` 域都一并检查才行。
3. 每个节点都带有一个高度为 1 层的后退指针，用于从表尾方向向表头方向迭代：当执行 [ZREVRANGE](http://redis.readthedocs.org/en/latest/sorted_set/zrevrange.html#zrevrange) 或 [ZREVRANGEBYSCORE](http://redis.readthedocs.org/en/latest/sorted_set/zrevrangebyscore.html#zrevrangebyscore) 这类以逆序处理有序集的命令时，就会用到这个属性。

跳跃表的结构定义：

```c
typedef struct zskiplist {

    // 头节点，尾节点
    struct zskiplistNode *header, *tail;

    // 节点数量
    unsigned long length;

    // 目前表内节点的最大层数
    int level;

} zskiplist;
```

跳跃表的节点定义：

```c
typedef struct zskiplistNode {

    // member 对象
    robj *obj;

    // 分值
    double score;

    // 后退指针
    struct zskiplistNode *backward;

    // 层
    struct zskiplistLevel {

        // 前进指针
        struct zskiplistNode *forward;

        // 这个层跨越的节点数量
        unsigned int span;

    } level[];

} zskiplistNode;
```

<img src="/images/Redis-skiplist/image-20200715223036068.png" alt="image-20200715223036068" style="zoom:67%;" />

上图就是跳跃列表的示意图，图中只画了3层，Redis 的跳跃表共有 64 层，容纳 2^64 个元素应该不成问题。

跳表的性质：

1. 由很多层结构组成
2. 每一层都是一个有序的链表
3. 最底层(Level 1) 的链表包含所有元素
4. 如果一个元素出现在 Level i 的链表中，则它在 Level i 之下的链表也都会出现。
5. 每个节点包含两个指针，一个指向同一链表中的下一个元素，一个指向下面一层的元素。

### 随机层数的设计

Redis 使用随机层数，解决插入、删除时，时间复杂度重新蜕化成O(n)的问题

它不要求上下相邻两层链表之间的节点个数有严格的对应关系，而是为每个节点随机出一个层数(level)。比如，一个节点随机出的层数是3，那么就把它链入到第1层到第3层这三层链表中。

<img src="/images/Redis-skiplist/image-20200715223056800.png" alt="image-20200715223056800" style="zoom:67%;" />

插入操作只需要修改插入节点前后的指针，而不需要对很多节点都进行调整。这就降低了插入操作的复杂度，这让它在插入性能上明显优于平衡树。

#### 随机层数的计算方式

执行插入操作时计算随机数的过程，是一个很关键的过程，它对skiplist的统计特性有着很重要的影响。这并不是一个普通的服从均匀分布的随机数，它的计算过程如下：

- 首先，每个节点肯定都有第1层指针（每个节点都在第1层链表里）。
- 如果一个节点有第i层(i>=1)指针（即节点已经在第1层到第i层链表中），那么它有第(i+1)层指针的概率为p。
- 节点最大的层数不允许超过一个最大值，记为MaxLevel。

这个计算随机层数的伪码如下所示：

```
randomLevel()
    level := 1
    // random()返回一个[0...1)的随机数
    while random() < p and level < MaxLevel do
        level := level + 1
    return level复制代码
```

randomLevel()的伪码中包含两个参数，一个是p，一个是MaxLevel。在Redis的skiplist实现中，这两个参数的取值为：

```
p = 1/4
MaxLevel = 32
```

#### skiplist的算法性能分析

根据前面randomLevel()的伪码，我们很容易看出，产生越高的节点层数，概率越低。

当skiplist中有n个节点的时候，它的总层数的概率均值是多少。这个问题直观上比较好理解。根据节点的层数随机算法，容易得出：

- 第1层链表固定有n个节点；
- 第2层链表平均有n*p个节点；
- 第3层链表平均有n*p2个节点；

计算很复杂，没有看懂，总结来说：平均时间复杂度为O(log n)

### rank排名计算

图中前向指针上面括号中的数字，表示对应的span的值。即当前指针跨越了多少个节点，这个计数不包括指针的起点节点，但包括指针的终点节点。

<img src="/images/Redis-skiplist/image-20200715223515614.png" alt="image-20200715223515614" style="zoom:67%;" />

举例：

在这个skiplist中查找score=89.0的元素（即Bob的成绩数据），在查找路径中，我们会跨域图中标红的指针，这些指针上面的span值累加起来，就得到了Bob的排名(2+2+1)-1=4（减1是因为rank值以0起始）。需要注意这里算的是从小到大的排名，而如果要算从大到小的排名，只需要用skiplist长度减去查找路径上的span累加值，即6-(2+2+1)=1。

通过这种方式就能得到一条O(log n)的查找路径

## 基本操作

### 查找操作

<img src="/images/Redis-skiplist/image-20200715223532032.png" alt="image-20200715223532032" style="zoom:67%;" />

实际上列表中是按照key进行排序的，查找过程也是根据key在比较。

### 插入操作

插入有两种情况 ① 小于等于原有的层数；② 大于原有的层数。需要做特殊处理。

## 跳跃表的应用

跳跃表在 Redis 的唯一作用， 就是实现有序集合。

## Redis为什么用skiplist而不用平衡树？

Redis的作者 @antirez 从内存占用、对范围查找的支持和实现难易程度做了解释。[原文](https://news.ycombinator.com/item?id=1171423)：

> There are a few reasons:
>
> 1) They are not very memory intensive. It's up to you basically. Changing parameters about the probability of a node to have a given number of levels will make then less memory intensive than btrees.
>
> 2) A sorted set is often target of many ZRANGE or ZREVRANGE operations, that is, traversing the skip list as a linked list. With this operation the cache locality of skip lists is at least as good as with other kind of balanced trees.
>
> 3) They are simpler to implement, debug, and so forth. For instance thanks to the skip list simplicity I received a patch (already in Redis master) with augmented skip lists implementing ZRANK in O(log(N)). It required little changes to the code.

总结来说：

* 不需要太多内存
* 范围查询和平衡树一样好
* 容易实现和调试

## 参考

[Redis设计与实现](https://book.douban.com/subject/25900156/)

[Redis 为什么用跳表而不用平衡树？](https://juejin.im/post/57fa935b0e3dd90057c50fbc#heading-0)


