# ﻿Redis数据类型总结


## redisObject 数据结构

`redisObject` 是 Redis 类型系统的核心， 数据库中的每个键、值，以及 Redis 本身处理的参数， 都表示为这种数据类型。

下图展示了 `redisObject` 、Redis 所有数据类型、以及 Redis 所有编码方式（底层实现）三者之间的关系：

<img src="C:\Users\mashuaisen\OneDrive\typora-user-images\Redis对象处理机制\image-20200714164101979.png" alt="image-20200714164101979" style="zoom: 50%;" />

### 引用计数以及对象的销毁

Redis 的对象系统使用了[引用计数](http://en.wikipedia.org/wiki/Reference_counting)技术来负责维持和销毁对象， 它的运作机制如下：

- 每个 `redisObject` 结构都带有一个 `refcount` 属性，指示这个对象被引用了多少次。
- 当新创建一个对象时，它的 `refcount` 属性被设置为 `1` 。
- 当对一个对象进行共享时，Redis 将这个对象的 `refcount` 增一。
- 当使用完一个对象之后，或者取消对共享对象的引用之后，程序将对象的 `refcount` 减一。
- 当对象的 `refcount` 降至 `0` 时，这个 `redisObject` 结构，以及它所引用的数据结构的内存，都会被释放。

## 五种数据类型

### 字符串

`REDIS_STRING` （字符串）是 Redis 使用得最为广泛的数据类型， 数据库中的所有键， 以及执行命令时提供给 Redis 的参数， 都是用这种类型保存的。

在 Redis 中， 只有能表示为 `long` 类型的值， 才会以整数的形式保存， 其他类型的整数、小数和字符串， 都是用 `sdshdr` 结构来保存。

<img src="J:/mind/blog/test/cool-sen.github.io.myBlog/static/images/Redis%E6%95%B0%E6%8D%AE%E7%B1%BB%E5%9E%8B%E6%80%BB%E7%BB%93/image-20200715224515345.png" alt="image-20200715224515345" style="zoom:50%;" />![image-20200715224534651](/images/Redis-Summary%20of%20data%20structure/image-20200715224534651.png)

### 哈希表

`REDIS_HASH` （哈希表）是 [HSET](http://redis.readthedocs.org/en/latest/hash/hset.html#hset) 、 [HLEN](http://redis.readthedocs.org/en/latest/hash/hlen.html#hlen) 等命令的操作对象， 它使用 `REDIS_ENCODING_ZIPLIST` 和 `REDIS_ENCODING_HT` 两种编码方式

<img src="J:/mind/blog/test/cool-sen.github.io.myBlog/static/images/Redis%E6%95%B0%E6%8D%AE%E7%B1%BB%E5%9E%8B%E6%80%BB%E7%BB%93/image-20200715224549814.png" alt="image-20200715224549814" style="zoom: 67%;" />

#### 字典编码的哈希表

程序将哈希表的键（key）保存为字典的键， 将哈希表的值（value）保存为字典的值，键和值都是字符串对象。

<img src="J:/mind/blog/test/cool-sen.github.io.myBlog/static/images/Redis%E6%95%B0%E6%8D%AE%E7%B1%BB%E5%9E%8B%E6%80%BB%E7%BB%93/image-20200715224620600.png" alt="image-20200715224620600" style="zoom:67%;" />

#### 压缩列表编码的哈希表

```
+---------+------+------+------+------+------+------+------+------+---------+
| ZIPLIST |      |      |      |      |      |      |      |      | ZIPLIST |
| ENTRY   | key1 | val1 | key2 | val2 | ...  | ...  | keyN | valN | ENTRY   |
| HEAD    |      |      |      |      |      |      |      |      | END     |
+---------+------+------+------+------+------+------+------+------+---------+
```

新添加的 key-value 对会被添加到压缩列表的表尾。

#### 编码的选择

创建空白哈希表时， 程序默认使用 ZIPLIST 编码， 当以下任何一个条件被满足时， 程序将编码从 ZIPLIST 切换为 HT（字典） ：

- 哈希表中某个键或某个值的长度大于 `server.hash_max_ziplist_value` （默认值为 `64` ）。
- 压缩列表中的节点数量大于 `server.hash_max_ziplist_entries` （默认值为 `512` ）。

### 列表

`REDIS_LIST` （列表）是 [LPUSH](http://redis.readthedocs.org/en/latest/list/lpush.html#lpush) 、 [LRANGE](http://redis.readthedocs.org/en/latest/list/lrange.html#lrange) 等命令的操作对象， 它使用 `ZIPLIST` 和 `LINKEDLIST` 这两种方式编码：

<img src="J:/mind/blog/test/cool-sen.github.io.myBlog/static/images/Redis%E6%95%B0%E6%8D%AE%E7%B1%BB%E5%9E%8B%E6%80%BB%E7%BB%93/image-20200715224646761.png" alt="image-20200715224646761" style="zoom: 67%;" />

#### 编码的选择

创建新列表时 Redis 默认使用 `REDIS_ENCODING_ZIPLIST` 编码， 当以下任意一个条件被满足时， 列表会被转换成 `REDIS_ENCODING_LINKEDLIST` 编码：

- 试图往列表新添加一个字符串值，且这个字符串的长度超过 `server.list_max_ziplist_value` （默认值为 `64` ）。
- `ziplist` 包含的节点超过 `server.list_max_ziplist_entries` （默认值为 `512` ）。

与哈希表使用压缩列表到字典的转换条件，很相似。

### 集合

`REDIS_SET` （集合）是 [SADD](http://redis.readthedocs.org/en/latest/set/sadd.html#sadd) 、 [SRANDMEMBER](http://redis.readthedocs.org/en/latest/set/srandmember.html#srandmember) 等命令的操作对象， 它使用 `INTSET` 和 `HT`（字典） 两种方式编码：

<img src="J:/mind/blog/test/cool-sen.github.io.myBlog/static/images/Redis%E6%95%B0%E6%8D%AE%E7%B1%BB%E5%9E%8B%E6%80%BB%E7%BB%93/image-20200715224710351.png" alt="image-20200715224710351" style="zoom:67%;" />

#### 编码的选择

第一个添加到集合的元素， 决定了创建集合时所使用的编码：

- 如果第一个元素可以表示为 `long long` 类型值（也即是，它是一个整数）， 那么集合的初始编码为 `REDIS_ENCODING_INTSET` 。
- 否则，集合的初始编码为 `REDIS_ENCODING_HT` 。

#### 编码的切换

如果一个集合使用 `REDIS_ENCODING_INTSET` 编码， 那么当以下任何一个条件被满足时， 这个集合会被转换成 `REDIS_ENCODING_HT` 编码：

- `intset` 保存的整数值个数超过 `server.set_max_intset_entries` （默认值为 `512` ）。
- 试图往集合里添加一个新元素，并且这个元素不能被表示为 `long long` 类型（也即是，它不是一个整数）。

### 有序集合

`REDIS_ZSET` （有序集合）是 [ZADD](http://redis.readthedocs.org/en/latest/sorted_set/zadd.html#zadd) 、 [ZCOUNT](http://redis.readthedocs.org/en/latest/sorted_set/zcount.html#zcount) 等命令的操作对象， 它使用 `ZIPLIST` 和 `SKIPLIST` 两种方式编码：

<img src="J:/mind/blog/test/cool-sen.github.io.myBlog/static/images/Redis%E6%95%B0%E6%8D%AE%E7%B1%BB%E5%9E%8B%E6%80%BB%E7%BB%93/image-20200715224744511.png" alt="image-20200715224744511" style="zoom:50%;" />

#### 编码的选择

如果第一个元素符合以下条件的话， 就创建一个 `REDIS_ENCODING_ZIPLIST` 编码的有序集：

- 服务器属性 `server.zset_max_ziplist_entries` 的值大于 `0` （默认为 `128` ）。
- 元素的 `member` 长度小于服务器属性 `server.zset_max_ziplist_value` 的值（默认为 `64` ）。

否则，程序就创建一个 `REDIS_ENCODING_SKIPLIST` 编码的有序集。

#### 编码的转换

对于一个 `REDIS_ENCODING_ZIPLIST` 编码的有序集， 只要满足以下任一条件， 就将它转换为 `REDIS_ENCODING_SKIPLIST` 编码：

* 新添加元素的 `member` 的长度大于服务器属性 `server.zset_max_ziplist_value` 的值（默认值为 `64` ）

- `ziplist` 所保存的元素数量超过服务器属性 `server.zset_max_ziplist_entries` 的值（默认值为 `128` ）

#### ZIPLIST 编码的有序集

每个有序集元素以两个相邻的 `ziplist` 节点表示， 第一个节点保存元素的 `member` 域， 第二个元素保存元素的 `score` 域。

多个元素之间按 `score` 值从小到大排序， 如果两个元素的 `score` 相同， 那么按字典序对 `member` 进行对比， 决定那个元素排在前面， 那个元素排在后面。

```
          |<--  element 1 -->|<--  element 2 -->|<--   .......   -->|

+---------+---------+--------+---------+--------+---------+---------+---------+
| ZIPLIST |         |        |         |        |         |         | ZIPLIST |
| ENTRY   | member1 | score1 | member2 | score2 |   ...   |   ...   | ENTRY   |
| HEAD    |         |        |         |        |         |         | END     |
+---------+---------+--------+---------+--------+---------+---------+---------+

score1 <= score2 <= ...
```

虽然元素是按 `score` 域有序排序的， 但对 `ziplist` 的节点指针只能线性地移动， 所以在 `REDIS_ENCODING_ZIPLIST` 编码的有序集中， 查找某个给定元素的复杂度为 O(N)O(N) 。

#### SKIPLIST 编码的有序集

`zset` 同时使用字典和跳跃表两个数据结构来保存有序集元素

其中， 元素的成员由一个 `redisObject` 结构表示， 而元素的 `score` 则是一个 `double` 类型的浮点数， 字典和跳跃表两个结构通过将指针共同指向这两个值来节约空间 （不用每个元素都复制两份）。

<img src="J:/mind/blog/test/cool-sen.github.io.myBlog/static/images/Redis%E6%95%B0%E6%8D%AE%E7%B1%BB%E5%9E%8B%E6%80%BB%E7%BB%93/image-20200715224806043.png" alt="image-20200715224806043" style="zoom: 67%;" />

通过使用字典结构， 并将 `member` 作为键， `score` 作为值， 有序集可以在 O(1)O(1) 复杂度内：

- 检查给定 `member` 是否存在于有序集（被很多底层函数使用）；
- 取出 `member` 对应的 `score` 值（实现 [ZSCORE](http://redis.readthedocs.org/en/latest/sorted_set/zscore.html#zscore) 命令）。

另一方面， 通过使用跳跃表， 可以让有序集支持以下两种操作：

- 在 O(logN)O(log⁡N) 期望时间、 O(N)O(N) 最坏时间内根据 `score` 对 `member` 进行定位（被很多底层函数使用）；
- 范围性查找和处理操作，这是（高效地）实现 [ZRANGE](http://redis.readthedocs.org/en/latest/sorted_set/zrange.html#zrange) 、 [ZRANK](http://redis.readthedocs.org/en/latest/sorted_set/zrank.html#zrank) 和 [ZINTERSTORE](http://redis.readthedocs.org/en/latest/sorted_set/zinterstore.html#zinterstore) 等命令的关键。

通过同时使用字典和跳跃表， 有序集可以高效地实现按成员查找和按顺序查找两种操作。

## 参考

[Redis设计与实现](https://book.douban.com/subject/25900156/)
