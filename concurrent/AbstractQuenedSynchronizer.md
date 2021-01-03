[TOC]
# 1.AQS简介
AbstractQuenedSynchronizer从名字就能看出AQS是一个_抽象的_基于队列的同步器。AQS本身是作为框架使用的，juc中很多同步工具比如ReentrantLock、CountDownLatch、Semphore、ReentrantReadWriteLock、FutureTask等类都是基于AQS来实现的，这几个工具都有作为同步工具的AQS的子类```Sync extends AbstractQueuedSynchronizer```。AQS提供了对内部资源state的原子性管理以及对线程调度的管理。
## 1.1内部类Node
## 1.2内部类ConditionObject

# 2.AQS源码
## 2.1acquire
## 2.2release
## 2.3acquireShared
## 2.4releaseShared