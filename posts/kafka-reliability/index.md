# Kafka如何保证数据可靠性


本文从不丢失，不重复有序性几个角度介绍数据的可靠性。

kafka 提供三种语义的传递：

- 至少一次 (at least once) 消息不会丢失 ack=all ，但是可能重复投递
- 至多一次 (at most once) 消息可能丢失，但是不会重复投递
- 精确一次 (Exactly Once) 消息不会丢失，也不会重复

## 如何保证消息不“丢”失

生产者，Broker，消费者都是有可能丢数据的。

### **生产端**

**生产端丢失数据**

即发送的数据根本没有保存到Broker端。出现这个情况的原因可能是，网络抖动，导致消息压根就没有发送到 Broker 端；也可能是消息本身不合格导致 Broker 拒绝接收（比如消息太大了，超过了 Broker 的承受能力）等等。

**生产端保证消息不丢失**

* 简单的send发送后不会去管它的结果是否成功，而callback能准确地告诉你消息是否真的提交成功了。一定要使用带有回调通知的 send 方法。
* broker一般不会有一个，我们就是要通过多Broker达到高可用的效果。设置 `acks = all`，表明所有副本 Broker 都要接收到消息，该消息才算是“已提交”，这样可以达到高可用的效果。
  - acks=1（默认）：当且仅当leader收到消息**返回commit确认信号**后认为发送成功。如果 leader 宕机，则会丢失数据。producer 等待 broker 的 ack，partition 的 leader 落盘成功后返回 ack，如果 follower 同步成功之前 leader 故障，那么就会丢失数据。
  - acks=0：producer发出消息即完成发送，**无需等待**来自 broker 的确认。这种情况下数据传输效率最高，但是数据可靠性确是最低的。
  - acks=-1（ALL）：发送端需要等待 **ISR 列表中所有列表都确认接收数据**后才算一次发送完成，可靠性最高，延迟也较大。如果 follower 同步完成后，broker 发送 ack 之前，leader 发生故障，producer 重新发送消息给新 leader 那么会造成数据重复。

**注意**

Acks=all 就可以代表数据一定不会丢失了吗?当然不是，如果你的 Partition 只有一个副本，也就是一个 Leader，任何 Follower 都没有，因为 ISR 里就一个 Leader，它接收完消息后宕机，也会导致数据丢失。 

所以说，这个 Acks=all，必须跟 ISR 列表里至少有 2 个以上的副本配合使用，起码是有一个 Leader 和一个 Follower 才可以

### **Broker端**

**Broker端丢失数据**

数据已经保存在broker端，但是数据却丢失了。出现这个的原因可能是，Broker机器down了，当然broker是高可用的，假如你的消息保存在 N 个 Kafka Broker 上，那么至少有 1 个存活就不会丢。

**Broker端保证消息不丢失**

kafka是有限度的保证消息不丢失，这里的限度，是至少一台存储了你消息的的broker。

关注一个leader选举的问题

kafka中有领导者副本（Leader Replica）和追随者副本（Follower Replica），而follower replica存在的唯一目的就是防止消息丢失，并不参与具体的业务逻辑的交互。只有leader 才参与服务，follower的作用就是充当leader的候补，平时的操作也只有信息同步。ISR也就是这组与leader保持同步的replica集合，我们要保证不丢消息，首先要保证ISR的存活（至少有一个备份存活），那存活的概念是什么呢，不仅需要机器正常，还需要跟上leader的消息进度，当达到一定程度的时候就会认为“非存活”状态。

### **消费端**

**消费端丢数据**

Consumer 程序有个“位移”的概念，表示的是这个 Consumer 当前消费到的 Topic 分区的位置。Kafka默认是自动提交位移的，这样可能会有个问题，假如你在pull(拉取)30条数据，处理到第20条时自动提交了offset，但是在处理21条的时候出现了异常，当你再次pull数据时，由于之前是自动提交的offset，所以是从30条之后开始拉取数据，这也就意味着21-30条的数据发生了丢失。

**消费端保证消息不丢失**

消费端保证不丢数据，最重要就是保证offset的准确性。我们能做的，就是确保消息消费完成再提交。Consumer 端有个参数 ，设置enable.auto.commit= false，并且采用手动提交位移的方式。如果在处理数据时发生了异常，那就把当前处理失败的offset进行提交(放在finally代码块中)注意一定要确保offset的正确性，当下次再次消费的时候就可以从提交的offset处进行再次消费。consumer在处理数据的时候失败了，其实可以把这条数据给缓存起来，可以是redis、DB、file等，也可以把这条消息存入专门用于存储失败消息的topic中，让其它的consumer专门处理失败的消息。

## 消息去重

kafka默认情况下，提供的是至少一次的可靠性保障(acks=all)。即broker保障已提交的消息的发送，但是遇上某些意外情况，如：网络抖动，超时等问题，导致Producer没有收到broker返回的数据ack，则Producer会继续重试发送消息，从而导致消息重复发送。

如果我们禁止Producer的失败重试发送功能,或者说不用等待服务器响应(acks=0)，消息要么写入成功，要么写入失败，但绝不会重复发送。这样就是最多一次的消息保障模式。

但对于消息组件，排除特殊业务场景，我们追求的一定是精确一次的消息保障模式。kafka通过幂等性（Idempotence）和事务（Transaction）的机制，提供了这种精确的消息保障。

### 幂等生产者

幂等生产者，说白了，就是你消息发送多次，对于系统也没影响。

Producer 默认不是幂等性的，但我们可以创建幂等性 Producer。指定 Producer 幂等性的方法很简单，仅需要设置一个参数即可，即 props.put(“enable.idempotence”, ture) ,被设置成 true 后，Producer 自动升级成幂等性 Producer，其他所有的代码逻辑都不需要改变。

Kafka 自动帮你做消息的重复去重。底层具体的原理很简单，就是经典的用空间去换时间的优化思路，**即在 Broker 端多保存一些字段。当 Producer 发送了具有相同字段值的消息后，Broker 能够自动知晓这些消息已经重复了**，于是可以在后台默默地把它们“丢弃”掉。

每个Producer在初始化的时候都会被分配一个唯一的PID，Producer向指定的Topic的特定Partition发送的消息都携带一个sequence number（简称seqNum），从零开始的单调递增的。

Broker会将Topic-Partition对应的seqNum在内存中维护，每次接受到Producer的消息都会进行校验；只有seqNum比上次提交的seqNum刚好大一，才被认为是合法的。比它大的，说明消息有丢失；比它小的，说明消息重复发送了。

以上说的这个只是针对单个Producer在一个session内的情况，假设Producer挂了，又重新启动一个Producer被而且分配了另外一个PID

注意以下问题：

1、它只能保证单分区上的幂等性，即一个幂等性 Producer 能够保证某个主题的一个分区上不出现重复消息，无法实现多个分区的幂等性。

2、它只能实现单会话上的幂等性，不能实现跨会话的幂等性。这里的会话，你可以理解为 Producer 进程的一次运行。当你重启了 Producer 进程之后，这种幂等性保证就丧失了。

那么如果我想实现多分区以及多会话上的消息无重复，应该怎么做呢？答案就是事务（transaction）或者依赖事务型 Producer。这也是幂等性 Producer 和事务型 Producer 的最大区别！

### 事务生产者

kafka的事务跟我们常见数据库事务概念差不多，也是提供经典的ACID，即原子性（Atomicity）、一致性 (Consistency)、隔离性 (Isolation) 和持久性 (Durability)。

事务Producer保证消息写入分区的原子性，即这批消息要么全部写入成功，要么全失败。

事务Producer保证消息写入分区的原子性，即这批消息要么全部写入成功，要么全失败。此外，Producer重启回来后，kafka依然保证它们发送消息的精确一次处理。

### 消费端

以上的事务性保证只是针对的producer端，对consumer端无法保证，有以下原因：

1. 压实类型的topics，有些事务消息可能被新版本的producer重写
2. 事务可能跨坐2个log segments，这时旧的segments可能被删除，就会丢消息
3. 消费者可能寻址到事务中任意一点，也会丢失一些初始化的消息
4. 消费者可能不会同时从所有的参与事务的TopicPartitions分片中消费消息

如果是消费kafka中的topic，并且将结果写回到kafka中另外的topic，
**可以将消息处理后结果的保存和offset的保存绑定为一个事务，这时就能保证
消息的处理和offset的提交要么都成功，要么都失败。**

如果是将处理消息后的结果保存到外部系统，这时就要用到两阶段提交（tow-phase commit），
但是这样做很麻烦，较好的方式是offset自己管理，将它和消息的结果保存到同一个地方，整体上进行绑定，

## 消息乱序

Kafka分布式的单位是partition，同一个partition用一个write ahead log组织，所以可以保证FIFO的顺序。不同partition之间不能保证顺序。

但是绝大多数用户都可以通过message key来定义，因为同一个key的message可以保证只发送到同一个partition，比如说key是user id，table row id等等，所以同一个user或者同一个record的消息永远只会发送到同一个partition上，保证了同一个user或record的顺序。当然，如果你有key skewness 就有些麻烦，需要特殊处理

因此，如果你们就像死磕kafka，但是对数据有序性有严格要求，那我建议：

1. 创建Topic只指定**1个partition**，这样的坏处就是磨灭了kafka最优秀的特性。

所以可以思考下是不是技术选型有问题， kafka本身适合与流式大数据量，要求高吞吐，对数据有序性要求不严格的场景。



## 参考

https://www.cnblogs.com/sddai/p/11340870.html

https://3gods.com/kafka/Kafka-Message-Delivery-Semantics.html#sec-1

http://bigdata-star.com/archives/1507

**我的博客即将同步至腾讯云+社区，邀请大家一同入驻：https://cloud.tencent.com/developer/support-plan?invite_code=2jqmbbj4xzaco**
