# Kafka消费者组


## 简介

消费者组是 Kafka 独有的概念，**消费者组是 Kafka 提供的可扩展且具有容错性的消费者机制**。

有多个消费者或消费者实例（Consumer Instance），它们共享一个公共的Group ID。组内的所有消费者协调在一起来消费订阅主题（Subscribed Topics）的所有分区（Partition）。

<img src="/images/Kafka-Consumer%20Group/image-20200713223952326.png" alt="image-20200713223952326" style="zoom:67%;" />

特性：

1. Consumer Group下可以有一个或多个Consumer实例。这里的实例可以是一个单独的进程，也可以是同
   一进程下的线程。在实际场景中，使用进程更为常见一些。
2. Group ID是一个字符串，在一个Kafka集群中，它标识唯一的一个Consumer Group。
3. Consumer Group下所有实例订阅的主题的单个分区，只能分配给组内的某个Consumer实例消费。这个
   分区当然也可以被其他的Group消费。

## 消费者组作用

> 传统的消息队列模型的缺陷在于消息一旦被消费，就会从队列中被删除，而且只能被下游的一个Consumer消费。这是它的一个特性，但这种模型的伸缩性（scalability）很差，因为下游的多个
> Consumer都要“抢”这个共享消息队列的消息。
>
> 发布/订阅模型倒是允许消息被多个Consumer消费，但它的问题也是伸缩性不高，因为每个订阅者都必须要订阅主题的所有分区。这种全量订阅的方式既不灵活，也会影响消息的真实投递效果。

Kafka仅仅使用Consumer Group这一种机制，却同时实现了传统消息引擎系统的两大模型：如果所有实例都属于同一个Group，那么它实现的就是消息队列模型；如果所有实例分别属于不同的Group，那么它实现的就是发布/订阅模型。

## 位移的管理

是一组KV对，Key是分区，V对应Consumer消费该分区的最新位移。

老版本的Consumer Group把位移保存在ZooKeeper中。最显而易见的好处就是减少了Kafka Broker端的状态保存开销。现在比较流行的是将服务器节点做成无状态的，这样可以自由地扩缩容，实现超强的伸缩性。

在新版本的Consumer Group中，采用了将位移保存在Kafka内部主题的方法。因为ZooKeeper这类元框架其实并不适合进行频繁的写更新，而Consumer Group的位移更新却是一个非常频繁的操作。这种大吞吐量的写操作会极大地拖慢ZooKeeper集群的性能。

## 重平衡Rebalance

Rebalance本质上是一种协议，规定了一个Consumer Group下的所有Consumer如何达成一致，来分配订阅Topic的每个分区。

### Rebalance的触发条件

1. 组成员数发生变更。比如有新的Consumer实例加入组或者离开组，抑或是有Consumer实例崩溃被“踢出”组。
2. 订阅主题数发生变更。Consumer Group可以使用正则表达式的方式订阅主题，比如
   consumer.subscribe(Pattern.compile(“t.*c”))就表明该Group订阅所有以字母t开头、字母c结尾的主题。在Consumer Group的运行过程中，你新创建了一个满足这样条件的主题，那么该Group就会发生Rebalance。
3. 订阅主题的分区数发生变更。Kafka当前只能允许增加一个主题的分区数。当分区数增加时，就会触发订阅该主题的所有Group开启Rebalance。

### 分配策略

Rebalance发生时，Group下所有的Consumer实例都会协调在一起共同参与。此时需要分配策略，来确定每个Consumer消费订阅主题的哪些分区。

### Rebalance的弊端

> 协调者Coordinator，专门为Consumer Group服务，负责为Group执行Rebalance以及提供位移管理和组成员管理等。Consumer端应用程序在提交位移时，其实是向Coordinator所在的Broker提交位移。同样地，当Consumer应用启动时，也是向Coordinator所在的Broker发送各种请求，然后由Coordinator负责执行消费者组的注册、成员管理记录等元数据管理操作。
>
> 所有Broker在启动时，都会创建和开启相应的Coordinator组件。Kafka为某个Consumer Group确定Coordinator所在的Broker的算法有2个步骤。
>
> 第1步：确定由位移主题的哪个分区来保存该Group数据：partitionId=Math.abs(groupId.hashCode() %offsetsTopicPartitionCount)。
> 第2步：找出该分区Leader副本所在的Broker，该Broker即为对应的Coordinator。

1. Rebalance影响Consumer端TPS。在Rebalance期间，Consumer会停下手头的事情，什么也干不了。
2. Rebalance很慢。
3. Rebalance效率不高。当前Kafka的设计机制决定了每次Rebalance时，Group下的所有成员都要参与进来，而且通常不会考虑局部性原理。

关于第3点，举个例子。比如一个Group下有10个成员，每个成员平均消费5个分区。假设现在有一个成员退出了，此时就需要开启新一轮的Rebalance，把这个成员之前负责的5个分区“转移”给其他员。

在默认情况下，每次Rebalance时，之前的分配方案都不会被保留。就拿刚刚这个例子来说，当Rebalance开始时，Group会打散这50个分区（10个成员 * 5个分区），由当前存活的9个成员重新分配它们。

显然这不是效率很高的做法。基于这个原因，社区于0.11.0.0版本推出了StickyAssignor，即有粘性的分区分配策略。所谓的有粘性，是指每次Rebalance时，该策略会尽可能地保留之前的分配方案，尽量实现分区分配的最小变动。不过有些遗憾的是，这个策略目前还有一些bug，而且需要升级到0.11.0.0才能使用，因此在实际生产环境中用得还不是很多。

因此尽量避免重平衡。

 
