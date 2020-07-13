# Redis数据类型-字典


字典（dictionary）， 又名映射（map）或关联数组（associative array）是一种抽象数据结构， 由一集键值对（key-value pairs）组成。

## 字典的应用

字典在 Redis 中的应用广泛。使用频率可以说和 SDS 以及双端链表不相上下

 字典的主要用途有以下两个：

1. 实现数据库键空间（key space）；
2. 用作 Hash 类型键的底层实现之一；

除此之外，带过期时间的 key 集合也是一个字典。zset 集合中存储 value 和 score 值的映射关系也是通过 dict 结构实现的。

### 1 .实现数据库键空间

Redis 是一个键值对数据库， 数据库中的键值对由字典保存： 每个数据库都有一个对应的字典， 这个字典被称之为键空间（key space）。

当用户添加一个键值对到数据库时（不论键值对是什么类型）， 程序就将该键值对添加到键空间； 当用户从数据库中删除键值对时， 程序就会将这个键值对从键空间中删除； 等等。

### 2.用作 Hash 类型键的底层实现

Redis 的 Hash 类型键使用以下两种数据结构作为底层实现:

1. 字典；
2. [压缩列表](https://redisbook.readthedocs.io/en/latest/compress-datastruct/ziplist.html#ziplist-chapter)；

因为压缩列表比字典更节省内存， 所以程序在创建新 Hash 键时， 默认使用压缩列表作为底层实现， 当有需要时， 程序才会将底层实现从压缩列表转换到字典。

## 字典的实现

实现字典的方法有很多种：

- 最简单的就是使用链表或数组，但是这种方式只适用于元素个数不多的情况下；
- 要兼顾高效和简单性，可以使用哈希表；
- 如果追求更为稳定的性能特征，并希望高效地实现排序操作的话，则可使用更为复杂的平衡树；

在众多可能的实现中， Redis 选择了高效、实现简单的哈希表，作为字典的底层实现。

### 1. 字典的定义

```c
/*
 * 每个字典使用两个哈希表，用于实现渐进式 rehash
 */
typedef struct dict {

    // 特定于类型的处理函数
    dictType *type;

    // 类型处理函数的私有数据
    void *privdata;

    // 哈希表（2 个）
    dictht ht[2];

    // 记录 rehash 进度的标志，值为 -1 表示 rehash 未进行
    int rehashidx;

    // 当前正在运作的安全迭代器数量
    int iterators;

} dict;
```

注意 `dict` 类型使用了两个指针，分别指向两个哈希表。

其中， 0 号哈希表（`ht[0]`）是字典主要使用的哈希表， 而 1 号哈希表（`ht[1]`）则只有在程序对 0 号哈希表进行 rehash 时才使用。

### 2. 哈希表实现

哈希表的定义：

```c
typedef struct dictht {

    // 哈希表节点指针数组（俗称桶，bucket）
    dictEntry **table;

    // 指针数组的大小
    unsigned long size;

    // 指针数组的长度掩码，用于计算索引值
    unsigned long sizemask;

    // 哈希表现有的节点数量
    unsigned long used;

} dictht;
```

`table` 属性是个数组， 数组的每个元素都是个指向 `dictEntry` 结构的指针。

每个 `dictEntry` 都保存着一个键值对， 以及一个指向另一个 `dictEntry` 结构的指针,哈希表节点定义：

```c
typedef struct dictEntry {

    // 键
    void *key;

    // 值
    union {
        void *val;
        uint64_t u64;
        int64_t s64;
    } v;

    // 链往后继节点
    struct dictEntry *next;

} dictEntry;
```

`dictht` [使用链地址法来处理键碰撞](http://en.wikipedia.org/wiki/Hash_table#Separate_chaining)： 当多个不同的键拥有相同的哈希值时，哈希表用一个链表将这些键连接起来。

下图展示了一个由 `dictht` 和数个 `dictEntry` 组成的哈希表例子：

<img src="/images/Redis-dict/image-20200713222219125.png" alt="image-20200713222219125" style="zoom:50%;" />

再加上之前列出的 `dict` 类型，整个字典结构可以表示如下：

<img src="/images/Redis-dict/image-20200713222447854.png" alt="image-20200713222447854" style="zoom:50%;" />

在上图的字典示例中， 字典虽然创建了两个哈希表， 但正在使用的只有 0 号哈希表， 这说明字典未进行 rehash 状态。

### 3. 哈希算法

Redis 目前使用两种不同的哈希算法：

1. MurmurHash2 32 bit 算法：这种算法的分布率和速度都非常好， 具体信息请参考 MurmurHash 的主页： http://code.google.com/p/smhasher/ 。
2. 基于 djb 算法实现的一个大小写无关散列算法：具体信息请参考 http://www.cse.yorku.ca/~oz/hash.html 。

使用哪种算法取决于具体应用所处理的数据：

- 命令表以及 Lua 脚本缓存都用到了算法 2 。
- 算法 1 的应用则更加广泛：数据库、集群、哈希键、阻塞操作等功能都用到了这个算法。

## 添加键值对到字典

​	根据字典所处的状态， 将给定的键值对添加到字典可能会引起一系列复杂的操作：

- 如果字典为未初始化（即字典的 0 号哈希表的 `table` 属性为空），则程序需要对 0 号哈希表进行初始化；
- 如果在插入时发生了键碰撞，则程序需要处理碰撞，使用链地址法来解决键冲突的问题；
- 如果插入新元素，使得字典满足了 rehash 条件，则需要启动相应的 rehash 程序；

整个添加流程可以用下图表示：

<img src="/images/Redis-dict/image-20200713223119823.png" alt="image-20200713223119823" style="zoom: 80%;" />

接下来重点介绍，添加新键值对时触发了 rehash 操作

### Rehash 触发条件

为了在字典的键值对不断增多的情况下保持良好的性能， 字典需要对所使用的哈希表（`ht[0]`）进行 rehash 操作： 在不修改任何键值对的情况下，对哈希表进行扩容， 尽量将比率维持在 1:1 左右。

`dictAdd` 在每次向字典添加新键值对之前， 都会对哈希表 `ht[0]` 进行检查， 对于 `ht[0]` 的 `size` 和 `used` 属性， 如果它们之间的比率 `ratio = used / size` 满足以下任何一个条件的话，rehash 过程就会被激活：

1. 自然 rehash ： `ratio >= 1` ，且变量 `dict_can_resize` 为真。
2. 强制 rehash ： `ratio` 大于变量 `dict_force_resize_ratio` （目前版本中， `dict_force_resize_ratio` 的值为 `5` ）。

> `dict_can_resize` 为假？
>
> 当 Redis 使用子进程对数据库执行后台持久化任务时（比如执行 `BGSAVE` 或 `BGREWRITEAOF` 时）， 为了最大化地利用系统的 [copy on write](http://en.wikipedia.org/wiki/Copy-on-write) 机制， 程序会暂时将 `dict_can_resize` 设为假， 避免执行自然 rehash ， 从而减少程序对内存的触碰（touch）。当持久化任务完成之后， `dict_can_resize` 会重新被设为真。
>
> 另一方面， 当字典满足了强制 rehash 的条件时， 即使 `dict_can_resize` 不为真（有 `BGSAVE` 或 `BGREWRITEAOF` 正在执行）， 这个字典一样会被 rehash 。

### Rehash 执行过程

字典的 rehash 操作实际上就是执行以下任务：

1. 创建一个比 `ht[0]->table` 更大的 `ht[1]->table` ；
2. 将 `ht[0]->table` 中的所有键值对迁移到 `ht[1]->table` ；
3. 将原有 `ht[0]` 的数据清空，并将 `ht[1]` 替换为新的 `ht[0]` ；

#### 1. 开始 rehash

1. 设置字典的 `rehashidx` 为 `0` ，标识着 rehash 的开始；

2. 为 `ht[1]->table` 分配空间，大小至少为 `ht[0]->used` 的两倍

   <img src="/images/Redis-dict/image-20200713223149728.png" alt="image-20200713223149728" style="zoom:67%;" />

#### 2. Rehash 进行中

 `ht[0]->table` 的节点会被逐渐迁移到 `ht[1]->table` ， 因为 rehash 是分多次进行的（细节在下一节解释）， 字典的 `rehashidx` 变量会记录 rehash 进行到 `ht[0]` 的哪个索引位置上。

注意使用的是渐进式 rehash。

<img src="/images/Redis-dict/image-20200713223219799.png" alt="image-20200713223219799" style="zoom:67%;" />

#### 3. Rehash 完毕

在 rehash 的最后阶段，程序会执行以下工作：

1. 释放 `ht[0]` 的空间；
2. 用 `ht[1]` 来代替 `ht[0]` ，使原来的 `ht[1]` 成为新的 `ht[0]` ；
3. 创建一个新的空哈希表，并将它设置为 `ht[1]` ；
4. 将字典的 `rehashidx` 属性设置为 `-1` ，标识 rehash 已停止；

<img src="/images/Redis-dict/image-20200713223256516.png" alt="image-20200713223256516" style="zoom:67%;" />

### 渐进式 rehash

#### 1. 为何采用渐进式rehash？

因为 redis 是单进程服务，所以当数据量很大的时候，扩容/缩容这些内存操作，涉及到新内存重新分配，数据拷贝。当数据量大的时候，会导致系统卡顿，必然会影响服务质量。

 Redis 使用了渐进式（incremental）的 rehash 方式： 通过将 rehash 分散到多个步骤中进行， 从而避免了集中式的计算。

#### 2. 渐进式rehash措施

在哈希表进行 rehash 时， 字典还会采取一些特别的措施， 确保 rehash 顺利、正确地进行：

- 添加时，新的节点会直接添加到 `ht[1]` 而不是 `ht[0]` ，这样保证 `ht[0]` 的节点数量在整个 rehash 过程中都只减不增。
- 因为在 rehash 时，字典会同时使用两个哈希表，所以在这期间的所有查找、删除等操作，除了在 `ht[0]` 上进行，还需要在 `ht[1]` 上进行。

> 渐进式 rehash 主要由 `_dictRehashStep` 和 `dictRehashMilliseconds` 两个函数进行：
>
> - `_dictRehashStep` 用于对数据库字典、以及哈希键的字典进行被动 rehash ；
> - `dictRehashMilliseconds` 则由 Redis 服务器常规任务程序（server cron job）执行，用于对数据库字典进行主动 rehash ；
>
> 在 rehash 开始进行之后（`d->rehashidx` 不为 `-1`）， 每次执行一次添加、查找、删除操作， `_dictRehashStep` 都会被执行一次。每次执行 `_dictRehashStep` ， `ht[0]->table` 哈希表第一个不为空的索引上的所有节点就会全部迁移到 `ht[1]->table` 。
>
> 当 Redis 的服务器常规任务执行时， `dictRehashMilliseconds` 会被执行， 在规定的时间内， 尽可能地对数据库字典中那些需要 rehash 的字典进行 rehash ， 从而加速数据库字典的 rehash 进程（progress）。



## 字典的收缩

场景：如果哈希表的可用节点数比已用节点数大很多的话， 那么也可以通过对哈希表进行 rehash 来收缩（shrink）字典。

步骤：

1. 创建一个比 `ht[0]->table` 小的 `ht[1]->table` ；
2. 将 `ht[0]->table` 中的所有键值对迁移到 `ht[1]->table` ；
3. 将原有 `ht[0]` 的数据清空，并将 `ht[1]` 替换为新的 `ht[0]` ；

何时收缩：当字典的填充率低于 10% 时， 程序就可以对这个字典进行收缩操作了， 每次从字典中删除一个键值对，如果字典达到了收缩的标准， 程序将立即对字典进行收缩。

字典收缩和扩展的区别：

- 字典的扩展操作是自动触发的（不管是自动扩展还是强制扩展）；
- 而字典的收缩操作则是由程序手动执行。


