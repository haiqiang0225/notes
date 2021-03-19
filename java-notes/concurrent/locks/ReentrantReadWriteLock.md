[toc]

# ReentrantReadWriteLock

> *本文源码基于JDK8*。因为本人水平有限，错误和不足之处在所难免，欢迎指出错误和不足之处，一起进步。
>
> 阅读这篇文章需要对AQS有一定的了解，虽然本篇文章大致介绍了AQS但还是强烈建议先去大致了解下AQS再来阅读本篇文章。
>
> 本篇文章中没怎么有对AQS的介绍，后续准备出一篇简单介绍AQS的博客，帮助想了解这些同步工具的实现但又不想去深挖AQS的小伙伴。

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
// 在获取共享锁增加共享部分计数时会有用
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

// 内部类，用于保存计数
static final class HoldCounter {
    int count = 0;
    // Use id, not reference, to avoid garbage retention
    // 使用id而不是引用，防止因为该类的强引用而产生无法回收线程的情况
    final long tid = getThreadId(Thread.currentThread());
}

// ThreadLocal的子类，用于绑定线程与计数器HoldCounter实例
static final class ThreadLocalHoldCounter
    extends ThreadLocal<HoldCounter> {
    public HoldCounter initialValue() {
        return new HoldCounter();
    }
}
// ThreadLocal的子类引用，用于操纵与线程绑定的HoldCounter实例
private transient ThreadLocalHoldCounter readHolds;

// 最为一种优化来使用，保存最后一次成功获得读锁的线程的计数器
// 可以省去ThreadLocalMap中查找HoldCounter实例的开销
// 因为只是作为优化，所以不用volatile来修饰，并且HoldCounter中
// 也保存着绑定线程的ID，因此也不需要用volatile来修饰，只有
// 绑定的那个线程才会使用到cachedHoldCounter
private transient HoldCounter cachedHoldCounter;

// firstReader记录了最后一个把共享部分的计数从0修改为1的线程
// 当这个线程完全释放读锁后会把firstReader设置为null
// firstReader也是作为一种优化来使用，当只有一个线程获得读锁时
// 开销会非常低，比如持有写锁的线程又获得读锁的情况
// firstReader指向的线程是不会在readHolds（其中线程绑定的计数器）
// 中存放计数的
private transient Thread firstReader = null;
// 用于保存firstReader指向的线程的读锁重入次数
private transient int firstReaderHoldCount;
```

我们来梳理一下：

1. Sync是AQS的子类，在此类中，把32位的state分为两部分，高16位用来存放读锁计数，低16位用来存放写锁计数，并且定义了多个用于获取、修改两部分的变量```SHARED_SHIFT、SHARED_UNIT 、MAX_COUNT、EXCLUSIVE_MASK```和两个分别用于获取读锁计数和写锁计数的方法```sharedCount```和```exclusiveCount```。

2. 内部类```HoldCounter```用于保存线程在读锁上的重入数，这个类的实例通过```ThreadLocal```的子类```ThreadLocalHoldCounter```（```readHolds```）来与线程绑定。

3. ```cachedHoldCounter```用来保存最后一个成功获取读锁的线程的引用，作用是可以省去在```ThreadLocalMap```中查找计数器的开销。

4. ```firstReader```和```firstReaderHoldCount```用来保存最后一个把共享部分计数从0改为1的线程的引用和读锁重入计数，可以优化只有单个线程获得读锁时的开销

我们知道，使用AQS构建同步工具需要重写以下几个方法：

1. 独占模式下需要重写：
   - ```protected boolean tryAcquire(int arg)```,获取资源成功则返回true，否则返回false
   - ```protected boolean tryRelease(int arg)```，释放资源成功返回true，否则返回false
   - ```protected boolean isHeldExclusively()```这个方法只有需要用到Condition的时候才需要重写，如果当前线程已经获得锁返回true，否则返回false
2. 共享模式下需要重写：
   - ```protected int tryAcquireShared(int arg)```，返回值为本次获取成功之后仍剩余的资源数目
   - ```protected boolean tryReleaseShared(int arg) ```，释放资源成功返回true，否则返回false

而很显然,```ReentrantReadWriteLock```使用了这俩种模式:读锁使用AQS共享模式实现,写锁使用独占模式实现。接下来我们通过源码看一下```ReentrantReadWriteLock```是如何做到借助AQS实现可重入、读写分离、不支持锁升级、支持锁降级等功能的吧。

先从独占模式看起：

#### tryAcquire方法：

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

独占模式下获取资源的逻辑是：

1. 先判断是否有线程持有读锁（不管是不是当前线程），如果有，获取失败。（不允许锁升级）
2. 没有线程持有读锁，但有线程持有写锁，判断一下持有写锁的线程是否是本线程、是否是重入的逻辑，不是的话，获取失败。
3. 在1和2都不满足且```state```不为0的情况即没有读锁但是有写锁，并且写锁的持有者是本线程，那么就是重入逻辑了，简单修改state就行了，CAS都不需要。
4. 最后一种情况就是```state```为0了，这说明前面读取```state```的时候是没有任何线程持有锁的，同```ReentrantLock```一样，这种情况下能否直接用CAS来尝试获得锁就需要看锁是公平模式还是非公平模式了。因为```state```等于0和是否有线程在AQS的同步队列等待是两回事，所以公平模式下需要判断是否有线程在排队，有的话当前线程是不能执行CAS的。而在非公平模式下，就可以尝试CAS来获取锁了，CAS成功则设置自己为独占线程，失败的话进入AQS同步队列排队。

其中，```writerShouldBlock()```实现如下，因为代码比较简单，因此我们不妨在这里先把```FairSync```和```NonfairSync```的代码看一下。

```java
static final class FairSync extends Sync {
    private static final long serialVersionUID = -2274990926593161451L;
    
    // 公平模式下，如果有比当前线程等待时间更长的线程，那么当前线程需要阻塞
    final boolean writerShouldBlock() {
        return hasQueuedPredecessors();
    }
    // 同上
    final boolean readerShouldBlock() {
        return hasQueuedPredecessors();
    }
}

static final class NonfairSync extends Sync {
    private static final long serialVersionUID = -8159625535654395037L;
    // 非公平模式下，获取读锁不需要阻塞（读多写少）
    final boolean writerShouldBlock() {
        return false; // writers can always barge
    }
    
    // 获取读锁是否要阻塞取决于在同步队列中排队的第一个线程是否是独占模式的
    // 如果是的话，当前线程需要阻塞，这是为了防止写线程一直无法获得写锁而导致线程饥饿
    final boolean readerShouldBlock() {
        /* As a heuristic to avoid indefinite writer starvation,
         * block if the thread that momentarily appears to be head
         * of queue, if one exists, is a waiting writer.  This is
         * only a probabilistic effect since a new reader will not
         * block if there is a waiting writer behind other enabled
         * readers that have not yet drained from the queue.
         */
        return apparentlyFirstQueuedIsExclusive();
    }
}

public final boolean hasQueuedPredecessors() {
    // The correctness of this depends on head being initialized
    // before tail and on head.next being accurate if the current
    // thread is first in queue.
    Node t = tail; // Read fields in reverse initialization order
    Node h = head;
    Node s;
    return h != t &&
        ((s = h.next) == null || s.thread != Thread.currentThread());
}

final boolean apparentlyFirstQueuedIsExclusive() {
    Node h, s;
    return (h = head) != null &&
        (s = h.next)  != null &&
        !s.isShared()         &&
        s.thread != null;
}
```

了解了独占模式下怎么获取资源之后我们再来看一下怎么释放资源（写锁）：

#### tryRelease：

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

因为是独占模式，所以释放资源的代码不需要同步，非常简单。

下面再看一下共享模式下是怎么获取资源（读锁）的：

#### tryAcquireShared：

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
    // 在一定程度上防止等待获取写锁的线程饥饿（因为是非公平模式，新来的线程
    // 可能会直接获得读锁），并且这个锁使用场景应该是读多写少，大量的读请求
    // 到来就可能会导致饥饿
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

共享模式下获得读锁的步骤如下：

1. 如果有写锁存在，判断一下是否是当前线程持有写锁，不是的话，获取读锁失败。（允许锁降级）

2. 没有写锁或者写锁被当前线程所持有的话，先判断获取读锁的操作是否要阻塞，是的话就进入```fullTryAcquireShared```方法中。

3. 不需要阻塞的话就尝试CAS的把共享部分的计数加1，表示获得了一次读锁，CAS失败的话也进入```fullTryAcquireShared```方法。

4. CAS成功的线程即获得读锁成功的线程，分不同的情况使自己对应的计数加1即可，这里不展开讨论。

可以看到，需要阻塞的线程、CAS失败的线程都进入了```fullTryAcquireShared```方法，这个方法中定义了读锁如何实现可重入的，是```tryAcquireShared```的“完全体”。

```fullTryAcquireShared```源码：

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
        // 因为这里是自旋，所以需要添加上这个判断，不添加这个判断的话
        // 考虑这种情况：当前线程读取的c，下面执行CAS的时候发现c变了，
        // CAS失败（这次失败是由于某个线程获得了写锁导致state改变），
        // 再次上来执行的时候又获得了c，这次c一直没改变，CAS成功，但是
        // 实际上已经有线程获得写锁了，所以在获取c之后要判断一次，确保没有写锁
        if (exclusiveCount(c) != 0) {
            // 如果这个if判定失败了，说明独占锁是被当前线程持有的
            // 该线程是可以获得读锁的，如果这里阻塞线程的话，就会造成死锁
            if (getExclusiveOwnerThread() != current)
                return -1;
            // else we hold the exclusive lock; blocking here
            // would cause deadlock.
            
        // 执行到这里，说明上面获取state的时刻是没有独占锁的，至于需不需要阻塞
        // 线程，需要看具体实现了，也就是前面的公平和非公平模式
        // 需要注意的是，读锁重入的线程是不会被阻塞的
        } else if (readerShouldBlock()) {
            // Make sure we're not acquiring read lock reentrantly
            // 已经获得读锁的线程再次获得读锁时不应该阻塞，否则就不满足可重入的语义了
            // 如果该线程是firstReader的话，不应该阻塞，因为如果线程是firstReader的
            // 话，说明线程已经获得读锁了，不管是只有读锁的情况还是持有写锁又获得读锁的情况
            if (firstReader == current) {
                // assert firstReaderHoldCount > 0;
            } else {
                if (rh == null) {
                    // 先尝试从缓存里查找自己获得读锁的次数，没有的话再从ThreadLocal
                    // 中查找
                    rh = cachedHoldCounter;
                    if (rh == null || rh.tid != getThreadId(current)) {
                        rh = readHolds.get();
                        // 说明线程还没有获得读锁
                        if (rh.count == 0)
                            // 暂时用不到的计数器就清除掉，等下一次获取时再生成一个
                            // 因为重写了initialValue()方法，所以不必担心下一次
                            // 获取时会获取到null
                            readHolds.remove();
                    }
                }
                // 不是重入，并且获取读锁失败的线程，则进入AQS排队
                if (rh.count == 0)
                    return -1;
            }
        }
        // 共享部分的计数已经达到最大值了，再次获取就会溢出
        if (sharedCount(c) == MAX_COUNT)
            throw new Error("Maximum lock count exceeded");
        // 跟tryAcquireShared方法中获取成功的逻辑相同
        // 1. 如果是把共享部分从0设置为1的线程，设置firstReader和对应的计数
        // 2. 如果是firstReader且共享部分不为0，则自增计数记录重入次数
        // 3. 不是firstReader的线程，先试着从缓存cachedHoldCounter中获取
        //    自己的计数，失败了的话再从线程绑定的readHolds中获取，
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

逻辑如下：

1. 先获取```state```，然后判断是否有写锁存在，确保不会出现注释中写的情况。

2. 如果```readerShouldBlock()```返回true，需要确保不是重入的线程，也就是说重入的线程是不会被阻塞的。不是重入的线程就会被阻塞。具体如何确保的请看注释。

3. ```readerShouldBlock()```返回false或者线程是重入的话，就执行CAS来修改state，成功的线程还是老样子，分不同情况来自增计数。

了解了尝试获取读锁的逻辑后,我们继续看释放的逻辑:

#### tryReleaseShared：

```java
protected final boolean tryReleaseShared(int unused) {
    Thread current = Thread.currentThread();
    // 是firstReader的话，修改firstReaderHoldCount
    if (firstReader == current) {
        // assert firstReaderHoldCount > 0;
        if (firstReaderHoldCount == 1)
            firstReader = null;
        else
            firstReaderHoldCount--;
    } else {
        // 获取线程读锁重入的次数
        HoldCounter rh = cachedHoldCounter;
        if (rh == null || rh.tid != getThreadId(current))
            rh = readHolds.get();
        int count = rh.count;
        if (count <= 1) {
            // 这次释放成功后线程理论上已经完全释放锁，因此要清除
            readHolds.remove();
            // 到这里还没有释放锁，如果重入次数已经小于等于0，则抛异常
            if (count <= 0)
                throw unmatchedUnlockException();
        }
        // 线程读锁重入计数减1
        --rh.count;
    }
    // CAS + 自旋的修改state，保证修改state的原子性
    // 修改成功后相当于真正的把锁释放了
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

释放的逻辑相对就简单了:

1. 先根据不同的情况来修改自己重入的计数。

2. 修改计数成功后，使用经典的CAS+自旋来修改state的值，保证原子性和一致性。

到这里，```ReentrantReadWriteLock```的源码基本就解读完了，剩下的代码都很简单了，直接贴下面了。

```java
public static class ReadLock implements Lock, java.io.Serializable {
    private static final long serialVersionUID = -5992448646407690164L;
    private final Sync sync;

    protected ReadLock(ReentrantReadWriteLock lock) {
        sync = lock.sync;
    }

    public void lock() {
        sync.acquireShared(1);
    }

    public void lockInterruptibly() throws InterruptedException {
        sync.acquireSharedInterruptibly(1);
    }

    public boolean tryLock() {
        return sync.tryReadLock();
    }

    public boolean tryLock(long timeout, TimeUnit unit)
            throws InterruptedException {
        return sync.tryAcquireSharedNanos(1, unit.toNanos(timeout));
    }

    public void unlock() {
        sync.releaseShared(1);
    }

    public Condition newCondition() {
        throw new UnsupportedOperationException();
    }

    public String toString() {
        int r = sync.getReadLockCount();
        return super.toString() +
            "[Read locks = " + r + "]";
    }
}
public static class WriteLock implements Lock, java.io.Serializable {
    private static final long serialVersionUID = -4992448646407690164L;
    private final Sync sync;

    protected WriteLock(ReentrantReadWriteLock lock) {
        sync = lock.sync;
    }
    
    public void lock() {
        sync.acquire(1);
    }

    public void lockInterruptibly() throws InterruptedException {
        sync.acquireInterruptibly(1);
    }

    public boolean tryLock( ) {
        return sync.tryWriteLock();
    }

    public boolean tryLock(long timeout, TimeUnit unit)
            throws InterruptedException {
        return sync.tryAcquireNanos(1, unit.toNanos(timeout));
    }

    public void unlock() {
        sync.release(1);
    }

    public Condition newCondition() {
        return sync.newCondition();
    }

    public String toString() {
        Thread o = sync.getOwner();
        return super.toString() + ((o == null) ?
                                   "[Unlocked]" :
                                   "[Locked by thread " + o.getName() + "]");
    }

    public boolean isHeldByCurrentThread() {
        return sync.isHeldExclusively();
    }
    
    public int getHoldCount() {
        return sync.getWriteHoldCount();
    }
}
```

可以看到，都是简单的调用```Sync```的方法。其中```Sync```中```tryWriteLock()```和```tryReadLock()```跟```tryAcquire```和```tryAcquireShared```的区别在于它们不会调用```writerShouldBlock()```、```readerShouldBlock```方法来判断是否需要阻塞，这和其它锁中```tryLock```的实现是一样的，就是线程不会去AQS中排队，失败就是失败了。这里不做详细讨论了。