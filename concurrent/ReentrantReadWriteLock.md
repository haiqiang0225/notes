[toc]

# ReentrantReadWriteLock

> *本文源码基于JDK8*。因为本人水平有限，错误和不足之处在所难免，欢迎指出错误和不足之处，一起进步。
>
> 阅读这篇文章需要对AQS有一定的了解，虽然本篇文章大致介绍了AQS但还是强烈建议先去大致了解下AQS再来阅读本篇文章。

## ReentrantReadWriteLock介绍

**ReentrantReadWriteLock**是juc中提供的同步工具，跟```ReentrantLock```一样，``ReentrantReadWriteLock``也是借助AQS实现的，也实现了公平模式和非公平模式，可重入。和```ReentrantLock```不同的是```ReentrantReadWriteLock```分别提供了**写锁**和**读锁**，其中写锁是独占锁，而读锁则是共享锁。

其中锁的关系如下：

1. 如果已经有线程持有读锁，那么所有线程都不能获得写锁，但是可以继续获得读锁
2. 如果已经有线程持有写锁，那么其他线程都不能再次获得任何锁，只有持有写锁的线程可以获得读锁和写锁
3. 允许锁降级，即线程可以：获得写锁 -> 获得读锁 -> 释放写锁 -> 释放读锁
4. 不允许锁升级，即线程不可以：获得读锁 -> 获得写锁 -> 释放读锁

```ReentrantReadWriteLock```主要适用于读多写少且要求数据强一致性的情景。

## ReentrantReadWriteLock使用

下面的例子简单的验证了几种获取锁的情况：

```java
@SuppressWarnings("all")
public class ReentrantReadWriteLockDemo {

    static final ReentrantReadWriteLock lock = new ReentrantReadWriteLock();

    static final Lock readLock = lock.readLock();

    static final Lock writeLock = lock.writeLock();


    /**
     * 输出：获得读锁
     * 线程会阻塞在writeLock.lock()这里，不允许锁升级
     */
    static void readThenWrite() {
        readLock.lock();
        System.out.println("获得读锁");
        writeLock.lock();
        System.out.println("获得读锁的情况下获得了写锁");
    }

     /**
     * 输出：
     * 获得写锁
     * 获得读锁
     * 释放写锁，锁降级
     * 释放读锁
     */
    static void lockDegradation() {
        writeLock.lock();
        System.out.println("获得写锁");
        readLock.lock();
        System.out.println("获得读锁");
        writeLock.unlock();
        System.out.println("释放写锁，锁降级");
        readLock.unlock();
        System.out.println("释放读锁");
    }

    /**
     * 输出：
     * 线程第一次获得写锁
     * 线程第二次成功获得写锁
     *
     * 允许重入
     */
    static void reGetWriteLock() {
        writeLock.lock();
        System.out.println("线程第一次获得写锁");
        writeLock.lock();
        System.out.println("线程第二次成功获得写锁");
    }

    /**
     * 输出：
     * 其他线程获得锁
     * 在其他锁获得读锁的情况下获得了写锁
     *
     * 没有后续输出，说明有其他线程持有读锁时是不能获得写锁的
     */
    static void otherThreadOwnedReadLock() {
        new Thread(() -> {
            readLock.lock();
            System.out.println(Thread.currentThread().getName() + "获得了锁");
        }).start();
        System.out.println("其他线程获得锁");
        writeLock.lock();
        System.out.println("在其他锁获得读锁的情况下获得了写锁");
    }
}
```

在看完上面的demo后我们来模拟有多个线程并发的读写某个变量的情况：

```java
import java.util.concurrent.TimeUnit;
import java.util.concurrent.locks.Lock;
import java.util.concurrent.locks.ReentrantReadWriteLock;

public class ReentrantReadWriteLockDemo2 {

    static final ReentrantReadWriteLock lock = new ReentrantReadWriteLock();

    static final Lock readLock = lock.readLock();

    static final Lock writeLock = lock.writeLock();

    static int state = 0;

    static class Reader implements Runnable {

        @Override
        public void run() {
            readLock.lock();
            try {
                System.out.println(Thread.currentThread().getName() + "读取到state为：" + state);
                TimeUnit.SECONDS.sleep(1);
            } catch (InterruptedException e) {
                e.printStackTrace();
            } finally {
                readLock.unlock();
            }
        }
    }

    static class Writer implements Runnable {

        @Override
        public void run() {
            writeLock.lock();
            try {
                state++;
                System.out.println(Thread.currentThread().getName() + "修改了state");
                TimeUnit.SECONDS.sleep(1);
            } catch (InterruptedException e) {
                e.printStackTrace();
            } finally {
                writeLock.unlock();
            }
        }
    }

    public static void main(String[] args) {
        for (int i = 0; i < 6; i++) {
            new Thread(new Reader()).start();
            new Thread(new Writer()).start();
        }
    }
}
```

运行这个例子某次的输出：

```java
Thread-0读取到state为：0
Thread-1修改了state
Thread-2读取到state为：1
Thread-3修改了state
Thread-6读取到state为：2
Thread-7修改了state
Thread-4读取到state为：3
Thread-10读取到state为：3
Thread-11修改了state
Thread-5修改了state
Thread-9修改了state
Thread-8读取到state为：6
```

可以看到，读线程读到的值和写线程写的次数是一致的，不会出现脏数据。

### 为什么不允许锁升级

这个其实很好理解，如果允许锁升级，那么读写锁就失去意义了。

```java
readLock.lock();
try {
    writeLock.lock();
    try {
        state++;
    } finally {
        writeLock.unlock();
    }
} finally {
    readLock.unlock();
}
```

上面的代码如果能正常执行的话，那么加锁和没加锁是一样的。就算是只有获得了读锁的那个线程能获得写锁也是不行的，因为很明显，这样会产生数据一致性问题，其他获得了读锁的线程可能看不到这次修改。

### 锁降级的作用

锁降级同样是为了保证数据一致性的一种手段。考虑下面的情况：

```java
public class RRWLockDemo3 {
    static final ReentrantReadWriteLock lock = new ReentrantReadWriteLock();

    static final Lock readLock = lock.readLock();

    static final Lock writeLock = lock.writeLock();

    static int state = 0;

    public static void main(String[] args) {
        Thread t1 = new Thread(() -> {
            int c = 0;
            writeLock.lock();
            try {
                state++;
                c = state;
            } finally {
                writeLock.unlock();
            }
            try {
                TimeUnit.SECONDS.sleep(1);
                System.out.println(c);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        });

        Thread t2 = new Thread(() -> {
            writeLock.lock();
            try {
                state++;
            } finally {
                writeLock.unlock();
            }
        });
        
        t1.start();
        t2.start();
    }
}
```

这里的代码模拟了两个线程，第一个线程增加state的之后继续使用state的本地存储，第二个线程获得写锁后修改了state，那么修改对第一个线程可能是不可见的。我们注意读变量前加读锁就可以避免这个问题（这里的例子可能不是很恰当）。

## ReentrantReadWriteLock源码分析

**ReentrantReadWriteLock**是借助AQS来实现的，内部通过组合了```Sync```、```FairSync```、```NonfairSync```这几个关键的类，搞懂了这几个类基本就相当于搞懂了```ReentrantReadWriteLock```了。

### Sync源码分析

内部类```Sync```继承了```AbstractQueuedSynchronizer```，AQS的```state```变量被划分成了两部分，其中<u>高16位</u>表示共享锁（读锁）占有的数量，<u>低16位</u>表示排它锁（写锁）占有的数量。下面是```Sync```中的成员变量。

```java
// 共享锁的偏移数，Java里int为32位，这里把int型的state分为了高16位和低16位两部分
static final int SHARED_SHIFT   = 16;
// 二进制：00000000 00000001 00000000 00000000
static final int SHARED_UNIT    = (1 << SHARED_SHIFT);
// 最大值显然是16位全1，从上面SHARED_UNIT的二进制不难看出，SHARED_UNIT-1就是最大值
static final int MAX_COUNT      = (1 << SHARED_SHIFT) - 1;
// 用来从state中取出独占锁的占有数量
// 二进制：00000000 00000000 11111111 11111111
// 显然只要用state与EXCLUSIVE_MASK执行按位与操作就能得到独占锁的占有数量
static final int EXCLUSIVE_MASK = (1 << SHARED_SHIFT) - 1;

// 下面两个Api分别用来获得共享锁和独占锁的数量
/** Returns the number of shared holds represented in count  */
static int sharedCount(int c)    { return c >>> SHARED_SHIFT; }
/** Returns the number of exclusive holds represented in count  */
static int exclusiveCount(int c) { return c & EXCLUSIVE_MASK; }
```

tryAcquire方法：

```java
protected final boolean tryAcquire(int acquires) {
    /*
     * Walkthrough:
     * 1. If read count nonzero or write count nonzero
     *    and owner is a different thread, fail.
     * 2. If count would saturate, fail. (This can only
     *    happen if count is already nonzero.)
     * 3. Otherwise, this thread is eligible for lock if
     *    it is either a reentrant acquire or
     *    queue policy allows it. If so, update state
     *    and set owner.
     */
    Thread current = Thread.currentThread();
    int c = getState();
    int w = exclusiveCount(c);
    if (c != 0) {
        // (Note: if c != 0 and w == 0 then shared count != 0)
        // state !=0 说明state的二进制不全为0，而独占部分的计数为0
        // 说明共享部分的计数不为0，这种情况显然不能获得独占锁，返回false
        // 另一种情况就是共享部分和独占部分的计数均不为0，这种情况就是
        // 已经获得写锁的线程又申请了写锁，所以这里要判断一下申请的线程
        // 是不是已经获得了锁的线程，不是的话也返回false
        if (w == 0 || current != getExclusiveOwnerThread())
            return false;
        // 获得写锁的数量已经超过了16位的最大值，抛出错误
        if (w + exclusiveCount(acquires) > MAX_COUNT)
            throw new Error("Maximum lock count exceeded");
        // Reentrant acquire
        // 能执行到这里说明独占部分和共享部分的计数都不为0，并且当前线程是
        // 已经获得了写锁的线程，再次获得写锁就是重入的逻辑了，设置state
        // 后返回true
        setState(c + acquires);
        return true;
    }
    // 运行到这里，说明 c == 0，独占部分和共享部分均为0，也就是说前面获取state的
    // 时候是没有线程获得读锁/写锁的，但是这和是否有线程在AQS中排队获得锁是没有关系的
    // 所以这里就涉及到是非公平锁还是公平锁的问题了，writerShouldBlock()实现了公平
    // 还是非公平锁。在非公平锁的实现中是直接返回false，而在公平锁的实现中是返回AQS中
    // 是否有线程在排队，这和ReentrantLock中的逻辑是一样的。
    // 简单来说，如果是非公平锁的话，会直接尝试CAS的修改state的值
    // 如果是公平锁的话，则是会先判断是否有线程在AQS中排队，如果有的话，自己也进入AQS排队
    // 如果没有的话才尝试CAS的修改state
    if (writerShouldBlock() ||
        !compareAndSetState(c, c + acquires))
        return false;
    // 执行到这里，当前线程成功获得写锁
    setExclusiveOwnerThread(current);
    return true;
}
```

tryRelease：

```java
protected final boolean tryRelease(int releases) {
    if (!isHeldExclusively())
        throw new IllegalMonitorStateException();
    int nextc = getState() - releases;
    boolean free = exclusiveCount(nextc) == 0;
    if (free)
        setExclusiveOwnerThread(null);
    // 因为是独占模式，所以并不需要并发控制，直接改就完了
    setState(nextc);
    return free;
}
```

tryAcquireShared：

```java
protected final int tryAcquireShared(int unused) {
    /*
     * Walkthrough:
     * 1. If write lock held by another thread, fail.
     * 2. Otherwise, this thread is eligible for
     *    lock wrt state, so ask if it should block
     *    because of queue policy. If not, try
     *    to grant by CASing state and updating count.
     *    Note that step does not check for reentrant
     *    acquires, which is postponed to full version
     *    to avoid having to check hold count in
     *    the more typical non-reentrant case.
     * 3. If step 2 fails either because thread
     *    apparently not eligible or CAS fails or count
     *    saturated, chain to version with full retry loop.
     */
    Thread current = Thread.currentThread();
    int c = getState();
    // 有写锁并且写锁不是被当前线程持有的情况下不能获得读锁
    // 不论公平模式还是非公平模式，这种情况下都是不能获得读锁的
    if (exclusiveCount(c) != 0 &&
        getExclusiveOwnerThread() != current)
        return -1;
    int r = sharedCount(c);
    // 1.先判断获得读锁的操作是否需要阻塞，这里也是有公平和非公平两种模式
    // 公平模式下会看一下AQS中是否有线程在排队，是的话返回true
    // 非公平模式和写锁直接返回false不需要阻塞不一样，会先判断一下AQS中排队
    // 的第一个线程是否是独占模式即等待写锁，是的话这次读锁要阻塞，这是为了
    // 在一定程度上防止等待获取写锁的线程饥饿
    // 2.如果获取读锁的操作不需要阻塞，在确保共享部分的计数不会溢出的情况下
    // 执行CAS来获取读锁
    if (!readerShouldBlock() &&
        r < MAX_COUNT &&
        // c + SHARED_UNIT 相当于共享部分的计数加1
        // SHARED_UNIT的二进制：00000000 00000001 00000000 00000000
        compareAndSetState(c, c + SHARED_UNIT)) {
        // 执行到这里线程已经成功获取读锁了
        // 如果之前共享部分的计数为0，说明是第一次获得读锁
        if (r == 0) {
            firstReader = current;
            firstReaderHoldCount = 1;
        } else if (firstReader == current) { 
            firstReaderHoldCount++;
        } else {
            // cachedHoldCounter作为一种优化，可以节省线程到ThreadLocalMap
            // 中查找holdCounter的开销
            HoldCounter rh = cachedHoldCounter;
            // cachedHoldCounter为空或者虽不为空，但缓存的并不是本线程的HoldCounter
            if (rh == null || rh.tid != getThreadId(current))
                // 从ThreadLocalMap中找到HoldCounter并缓存
                cachedHoldCounter = rh = readHolds.get();
            // 满足以下情况：
            // 1.cachedHoldCounter不为空且是缓存的是本线程的HoldCounter
            // 2.cachedHoldCounter的count为0，说明是第一次获得读锁
            else if (rh.count == 0)
                // 第一次获得读锁需要把HoldCounter的实例保存到线程绑定的readHolds中
                readHolds.set(rh);
            // 本线程持有的读锁计数加一
            rh.count++;
        }
        // 获取成功
        return 1;
    }
    // 如果执行下面的方法，有以下几种情况
    // 1. readerShouldBlock()返回了true，该次获得读锁的操作应该阻塞
    // 2. 共享部分的计数大于等于MAX_COUNT了
    // 3. 发生了竞争，导致该线程执行CAS失败
    return fullTryAcquireShared(current);
}
```

fullTryAcquireShared：

```java
final int fullTryAcquireShared(Thread current) {
    /*
     * This code is in part redundant with that in
     * tryAcquireShared but is simpler overall by not
     * complicating tryAcquireShared with interactions between
     * retries and lazily reading hold counts.
     */
    HoldCounter rh = null;
    for (;;) {
        int c = getState();
        // 写锁计数不为0且写锁不是当前线程持有，获取失败
        if (exclusiveCount(c) != 0) {
            // 如果这个if判定失败了，说明独占锁是被当前线程持有的
            // 如果这里阻塞线程的话，就会造成死锁，并且也失去了可重入的语义
            if (getExclusiveOwnerThread() != current)
                return -1;
            // else we hold the exclusive lock; blocking here
            // would cause deadlock.
            
        // 执行到这里，说明上面获取state的时刻是没有独占锁的，至于需不需要阻塞
        // 线程，需要看具体实现了，也就是前面的公平和非公平模式
        } else if (readerShouldBlock()) {
            // Make sure we're not acquiring read lock reentrantly
            if (firstReader == current) {
                // assert firstReaderHoldCount > 0;
            } else {
                if (rh == null) {
                    rh = cachedHoldCounter;
                    if (rh == null || rh.tid != getThreadId(current)) {
                        rh = readHolds.get();
                        if (rh.count == 0)
                            readHolds.remove();
                    }
                }
                if (rh.count == 0)
                    return -1;
            }
        }
        // 共享部分的计数已经达到最大值了，再次获取就会溢出
        if (sharedCount(c) == MAX_COUNT)
            throw new Error("Maximum lock count exceeded");
        // 跟tryAcquireShared方法中获取成功的逻辑相同
        if (compareAndSetState(c, c + SHARED_UNIT)) {
            if (sharedCount(c) == 0) {
                firstReader = current;
                firstReaderHoldCount = 1;
            } else if (firstReader == current) {
                firstReaderHoldCount++;
            } else {
                if (rh == null)
                    rh = cachedHoldCounter;
                if (rh == null || rh.tid != getThreadId(current))
                    rh = readHolds.get();
                else if (rh.count == 0)
                    readHolds.set(rh);
                rh.count++;
                cachedHoldCounter = rh; // cache for release
            }
            return 1;
        }
    }
}
```

tryReleaseShared：

```java
protected final boolean tryReleaseShared(int unused) {
    Thread current = Thread.currentThread();
    if (firstReader == current) {
        // assert firstReaderHoldCount > 0;
        if (firstReaderHoldCount == 1)
            firstReader = null;
        else
            firstReaderHoldCount--;
    } else {
        HoldCounter rh = cachedHoldCounter;
        if (rh == null || rh.tid != getThreadId(current))
            rh = readHolds.get();
        int count = rh.count;
        if (count <= 1) {
            readHolds.remove();
            if (count <= 0)
                throw unmatchedUnlockException();
        }
        --rh.count;
    }
    for (;;) {
        int c = getState();
        int nextc = c - SHARED_UNIT;
        if (compareAndSetState(c, nextc))
            // Releasing the read lock has no effect on readers,
            // but it may allow waiting writers to proceed if
            // both read and write locks are now free.
            return nextc == 0;
    }
}
```

