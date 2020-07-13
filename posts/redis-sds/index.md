# ﻿Redis数据类型-简单动态字符串


Sds （Simple Dynamic String，简单动态字符串）是 Redis 底层所使用的字符串表示， 几乎所有的 Redis 模块中都用了 sds。

## sds 的用途

Sds 在 Redis 中的主要作用有以下两个：

1. 实现字符串对象（StringObject）；
2. 在 Redis 程序内部用作 `char*` 类型的替代品；

### 1. 实现字符串对象

Redis 是一个键值对数据库（key-value DB）， 数据库的值可以是字符串、集合、列表等多种类型的对象， 而数据库的键则总是字符串对象。

### 2. 用 sds 取代 C 默认的 char* 类型

因为 `char*` 类型的功能单一， 抽象层次低， 并且不能高效地支持一些 Redis 常用的操作（比如追加操作和长度计算操作）， 所以在 Redis 程序内部， 绝大部分情况下都会使用 sds 而不是 `char*` 来表示字符串。

## Redis 中的字符串 设计

### 1. C 语言字符串缺点

在 C 语言中，字符串可以用一个 `\0` 结尾的 `char` 数组来表示。

比如说， `hello world` 在 C 语言中就可以表示为 `"hello world\0"` 。

这种简单的字符串表示，在大多数情况下都能满足要求，但是，它并不能高效地支持长度计算和追加（append）这两种操作：

- 每次计算字符串长度（`strlen(s)`）的复杂度为 θ(N)θ(N) 。
- 对字符串进行 N 次追加，必定需要对字符串进行 N 次内存重分配（`realloc`）。

Redis 的字符串表示还应该是[二进制安全的](http://en.wikipedia.org/wiki/Binary-safe)： 程序不应对字符串里面保存的数据做任何假设， 数据可以是以 `\0` 结尾的 C 字符串

考虑到这两个原因， Redis 使用 sds 类型替换了 C 语言的默认字符串表示： sds 既可高效地实现追加和长度计算， 同时是二进制安全的。

### 2. sds 的实现

```
typedef char *sds;

struct sdshdr {

    // buf 已占用长度
    int len;

    // buf 剩余可用长度
    int free;

    // 实际保存字符串数据的地方
    char buf[];
};
```

其中，类型 `sds` 是 `char *` 的别名（alias），而结构 `sdshdr` 则保存了 `len` 、 `free` 和 `buf` 三个属性。

* 通过 `len` 属性， `sdshdr` 可以实现复杂度为 θ(1) 的长度计算操作。
* 通过对 `buf` 分配一些额外的空间， 并使用 `free` 记录未使用空间的大小， `sdshdr` 可以让执行追加操作所需的内存重分配次数大大减少

### 3. 优化追加操作

内存分配策略

```
def sdsMakeRoomFor(sdshdr, required_len):

    # 预分配空间足够，无须再进行空间分配
    if (sdshdr.free >= required_len):
        return sdshdr

    # 计算新字符串的总长度
    newlen = sdshdr.len + required_len

    # 如果新字符串的总长度小于 SDS_MAX_PREALLOC
    # 那么为字符串分配 2 倍于所需长度的空间
    # 否则就分配所需长度加上 SDS_MAX_PREALLOC 数量的空间
    if newlen < SDS_MAX_PREALLOC:
        newlen *= 2
    else:
        newlen += SDS_MAX_PREALLOC

    # 分配内存
    newsh = zrelloc(sdshdr, sizeof(struct sdshdr)+newlen+1)

    # 更新 free 属性
    newsh.free = newlen - sdshdr.len

    # 返回
    return newsh
```

在目前版本的 Redis 中， `SDS_MAX_PREALLOC` 的值为 `1024 * 1024` 

总结：

* 当大小小于 `1MB` 的字符串执行追加操作时， `sdsMakeRoomFor` 就为它们分配多于所需大小一倍的空间
* 当字符串的大小大于 `1MB` ， 那么 `sdsMakeRoomFor` 就为它们额外多分配 `1MB` 的空间



分配策略浪费空间吗？

* 因为执行 [APPEND](http://redis.readthedocs.org/en/latest/string/append.html#append) 命令的字符串键数量通常并不多， 占用内存的体积通常也不大， 所以这一般并不算什么问题
* 另一方面， 如果执行 [APPEND](http://redis.readthedocs.org/en/latest/string/append.html#append) 操作的键很多， 而字符串的体积又很大的话， 那可能就需要修改 Redis 服务器， 让它定时释放一些字符串键的预分配空间， 从而更有效地使用内存。
