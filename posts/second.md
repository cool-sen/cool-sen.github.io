# Second

# AQS
## 简介
AQS其实就是一个可以给我们实现锁的框架

内部实现的关键是：先进先出的队列、state状态

定义了内部类ConditionObject

拥有两种线程模式

独占模式

共享模式

在LOCK包中的相关锁(常用的有ReentrantLock、 ReadWriteLock)都是基于AQS来构建


