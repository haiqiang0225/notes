[toc]

# ReentrantReadWriteLock

> *本文源码基于JDK8*。因为本人水平有限，错误和不足之处在所难免，欢迎指出错误和不足之处，一起进步。

## ReentrantReadWriteLock介绍

**ReentrantReadWriteLock**是juc中提供的同步工具，跟```ReentrantLock```一样，``ReentrantReadWriteLock``也是借助AQS实现的。和```ReentrantLock```不同的是```ReentrantReadWriteLock```分别提供了**写锁**和**读锁**，其中写锁是独占锁，而读锁则是共享锁。也就是说当有线程占有写锁时，其它线程无法获得任何锁，同时有其它线程占有任何锁时，线程将无法获得写锁，**也就是说如果有线程获得了ReentrantReadWriteLock的写锁，那么ReentrantReadWriteLock上有且只有这一个锁存在**。只存在读锁的情况下任意线程理论上都可以再次获得写锁。```ReentrantReadWriteLock```主要适用于读多写少但又要求数据强一致性的情景。

## ReentrantReadWriteLock使用

## ReentrantReadWriteLock源码分析

### Sync源码分析

