[TOC]
# 1.AQS简介
AbstractQuenedSynchronizer从名字就能看出AQS是一个_抽象的_基于队列的同步器。AQS本身是作为框架使用的，juc中很多同步工具比如ReentrantLock、CountDownLatch、Semphore、ReentrantReadWriteLock、FutureTask等类都是基于AQS来实现的，这几个工具都有作为同步工具的AQS的子类```Sync extends AbstractQueuedSynchronizer```。AQS提供了对内部资源state的原子性管理以及对线程调度的管理。
AQS内部主要有一个变量state、一个FIFO的CLH队列、内部类Node和ConditionObject
volatile的变量state：该变量可以理解为资源的数量，比如在ReentrantLock中该变量就可以表示为是否获得锁以及获得锁的次数。
先进先出（FIFO）的CLH队列：当线程尝试获取资源失败时就会加入到这个队列中，等待调度。
Node：用于封装线程，获取资源失败的线程会被封装为Node然后加入到CLH同步队列中。
ConditionObject：用于提供条件等待队列的支持，提供类似于Object类的wait和signal的api，内部维护了一个单链表，调用了await的线程会被加入到该链表中。

## 1.1内部类Node
## 1.2内部类ConditionObject

# 2.AQS源码
## 2.1acquire
## 2.2release
## 2.3acquireShared
## 2.4releaseShared