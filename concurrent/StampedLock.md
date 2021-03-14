[toc]

# StampedLock

## StampedLock介绍

**StampedLock**是JDK1.8中新增的同步工具，通过它的名字我们可能想到它和“戳”有关，这个后面再说。跟```ReentrantReadWriteLock```作用相似，适用于读多写少的场景。与```ReentrantReadWriteLock```的不同之处有以下几点：

1. ```StampedLock```提供了**乐观读锁、悲观读锁、写锁**三种锁，而```ReentrantReadWriteLock```只有悲观读锁和写锁两种模式。

2. ```StampedLock```是不可重入的，而```ReentrantReadWriteLock```是可重入的。

3. ```StampedLock```不是基于AQS的，```ReentrantReadWriteLock```是基于AQS的。```StampedLock```也是基于CLH队列的，CLH队列是一个在本地变量上自旋的锁实现，每个等待的线程不断轮询直接前驱的状态，如果前驱释放了锁，自己就尝试去获取锁。

```StampedLock```乐观读锁认为读的过程中不会发生写操作，因此并不加锁，只是在读完后验证一下读的过程中是否发生了写操作，如果发生了写操作，再进行同步来重新读取数据。这也是```StampedLock```跟```ReentrantReadWriteLock```最大的不同之处，```ReentrantReadWriteLock```当有线程持有读锁时是会阻塞当前获取写锁的线程，而```StampedLock```乐观读锁并不会阻塞其他线程。此外，```StampedLock```的悲观读锁跟```ReentrantReadWriteLock```相似，是与写锁互斥的。下面通过几个例子来看一下```StampedLock```如何使用。

## StampedLock使用

下面是jdk中提供的例子：

```java
class Point {
    private double x, y;
    private final StampedLock sl = new StampedLock();

    // 获取写锁（独占锁）
    void move(double deltaX, double deltaY) { // an exclusively locked method
        long stamp = sl.writeLock();
        try {
            x += deltaX;
            y += deltaY;
        } finally {
            sl.unlockWrite(stamp);
        }
    }

    // 乐观锁
    double distanceFromOrigin() { // A read-only method
        // tryOptimisticRead()返回一个“戳”
        long stamp = sl.tryOptimisticRead();
        double currentX = x, currentY = y;
        // 读取完成后判断一下上面获取的戳是否还有效
        if (!sl.validate(stamp)) {
            // 戳失效的话，改为使用悲观读锁
            stamp = sl.readLock();
            try {
                currentX = x;
                currentY = y;
            } finally {
                sl.unlockRead(stamp);
            }
        }
        // 即使上面获取悲观读锁了，但是这里currentX和currentY
        // 仍然有可能不是最新的，因为上面悲观读锁已经释放了，如果
        // 悲观读锁释放后又有线程修改了数据，那么本线程是感觉不到的
        // 个人感觉如果要求数据一致性比较强的话，应该在上面try块
        // 里返回计算结果，因为try块里的结果会先于finally块中锁的释放
        // jdk10中提供了新的api，这里下面再说
        return Math.sqrt(currentX * currentX + currentY * currentY);
    }

    // 将读锁升级为写锁
    void moveIfAtOrigin(double newX, double newY) { // upgrade
        // Could instead start with optimistic, not read mode
        long stamp = sl.readLock();
        try {
            while (x == 0.0 && y == 0.0) {
                // 尝试将读锁升级为写锁
                // 1. 如果当前线程拥有写锁，该方法会直接返回
                // 2. 如果当前线程拥有读锁，如果写锁可用，就将读锁释放并返回获得
                //    写锁的戳记
                // 3. 如果是乐观读锁，当写锁立即可用时，返回写戳记
                // 4. 其他情况返回0
                // ws如果不为0L，说明锁升级成功
                long ws = sl.tryConvertToWriteLock(stamp);
                if (ws != 0L) {
                    stamp = ws;
                    x = newX;
                    y = newY;
                    // 锁升级后不用再释放读锁了，直接unlock刚才
                    // 锁升级成功后返回的stamp即可
                    break;
                }
                else {
                    // 锁升级失败，先释放读锁再按正常流程去获得写锁
                    sl.unlockRead(stamp);
                    stamp = sl.writeLock();
                }
            }
        } finally {
            sl.unlock(stamp);
        }
    }
}
```

从上面的例子可以看到，```StampedLock```乐观读锁实际上并未加锁，也不会阻塞写锁的获取，获得乐观读锁后会返回一个戳记（stamp），在读取变量之后利用```StampedLock```提供的api来判断一下这个戳记是否还有效，有效的话说明读取过程中并没有发生写入，反之则表示读取变量的过程中发生了写入，需要我们根据我们的要求进行额外的同步。

jdk10中提供了新的api，如果你使用的是jdk10以上的jdk版本，对于乐观读锁的使用可以参考以下例子：

```java
// a read-only method
// upgrade from optimistic read to read lock
double distanceFromOrigin() {
    // 先尝试用乐观读锁
    long stamp = sl.tryOptimisticRead();
    try {
        for (; ; stamp = sl .readLock()) {
            // 如果乐观读锁获取失败，执行末尾循环体中获取悲观读锁的
            // 语句，锁从乐观读锁升级为悲观读锁
            if (stamp == 0L)
                continue;
            // possibly racy reads
            double currentX = x;
            double currentY = y;
            // 读取过程中发生了写入操作，同上操作一样，锁升级
            if (!sl.validate(stamp))
                continue;
            // 没有发生写入的话就返回
            return Math.hypot(currentX, currentY);
        }
    } finally {
        // jdk10提供的新api，可以判断获取锁后返回的戳记是否是由悲观读锁返回的
        if (StampedLock.isReadLockStamp(stamp))
            sl.unlockRead(stamp);
    }
}

// upgrade from optimistic read to write lock
void moveIfAtOrigin(double newX, double newY) {
    long stamp = sl.tryOptimisticRead();
    try {
        for (; ; stamp = sl.writeLock()) {
            // 获得乐观读锁失败的话，直接升级为写锁
            if (stamp == 0L)
                continue;
            // possibly racy reads
            double currentX = x;
            double currentY = y;
            // 读取的过程中发生了写入也升级为写锁
            if (!sl.validate(stamp))
                continue;
            if (currentX != 0.0 || currentY != 0.0)
                break;
            // 尝试转换为写锁
            stamp = sl.tryConvertToWriteLock(stamp);
            // 失败就进入正常获取写锁的流程
            if (stamp == 0L)
                continue;
            // exclusive access
            // 能执行到这里说明已经获得写锁了
            x = newX;
            y = newY;
            return;
        }
    } finally {
        // 判断是否是写锁的戳记，是的话才释放
        if (StampedLock.isWriteLockStamp(stamp))
            sl.unlockWrite(stamp);
    }
}
```

## StampedLock源码分析



```java

// jvm可用的cpu个数
private static final int NCPU = Runtime.getRuntime().availableProcessors();

/** Maximum number of retries before enqueuing on acquisition */
private static final int SPINS = (NCPU > 1) ? 1 << 6 : 0;

/** Maximum number of retries before blocking at head on acquisition */
private static final int HEAD_SPINS = (NCPU > 1) ? 1 << 10 : 0;

/** Maximum number of retries before re-blocking */
private static final int MAX_HEAD_SPINS = (NCPU > 1) ? 1 << 16 : 0;

// 二进制：0000 ... 0000 0111
private static final int OVERFLOW_YIELD_RATE = 7; // must be power 2 - 1

// 在溢出之前用于读计数器的比特位数，这里是7位
private static final int LG_READERS = 7;

// Values for lock state and stamp operations

// 跟ReentrantReadWriteLock类似，RUNIT用来增加读锁计数
private static final long RUNIT = 1L;
// 写锁标记位 0000 ... 1000 0000
private static final long WBIT  = 1L << LG_READERS;
// 读标记总位数 0000 ... 0111 1111
private static final long RBITS = WBIT - 1L;
// 读标记的最大值 0000 ... 0111 1110
private static final long RFULL = RBITS - 1L;
// 0000 ... 1111 1111  ABITS作为掩码使用，可以迅速的取出后8位，包括写锁
// 和读锁的所有比特位，因此可以用 ABITS & state 来判断是否有锁
private static final long ABITS = RBITS | WBIT;
// 1111 ... 1000 0000
private static final long SBITS = ~RBITS; // note overlap with ABITS

// Initial value for lock state; avoid failure value zero
// 0000 ... 0001 0000 0000
private static final long ORIGIN = WBIT << 1;

// Special value from cancelled acquire methods so caller can throw IE
private static final long INTERRUPTED = 1L;

// Values for node status; order matters
private static final int WAITING   = -1;
private static final int CANCELLED =  1;

// Modes for nodes (int not boolean to allow arithmetic)
private static final int RMODE = 0;
private static final int WMODE = 1;

/** Wait nodes */
static final class WNode {
    volatile WNode prev;
    volatile WNode next;
    volatile WNode cowait;    // list of linked readers
    volatile Thread thread;   // non-null while possibly parked
    volatile int status;      // 0, WAITING, or CANCELLED
    final int mode;           // RMODE or WMODE
    WNode(int m, WNode p) { mode = m; prev = p; }
}

/** CLH队列头结点的引用 */ 
private transient volatile WNode whead;
/** CLH队列尾结点的引用 */
private transient volatile WNode wtail;

// views
// 锁视图，StampedLock并没有实现java.util.concurrent.locks.Lock接口
// 下面的视图分别实现了Lock/ReadWriteLock接口，内部还是调用的StampedLock的方法
transient ReadLockView readLockView;
transient WriteLockView writeLockView;
transient ReadWriteLockView readWriteLockView;

/** Lock sequence/state */
private transient volatile long state;
// state中7位的读锁计数溢出后的计数用readerOverflow来记录，该变量不是
// volatile的，因此需要同步来保证一致性
/** extra reader count when state read count saturated */
private transient int readerOverflow;
```

### 内部重要属性

内部的属性比较多，我们先来梳理一下：

1. 内部维护一个```volatile```的变量```state```，用来记录锁的状态，跟```ReentrantReadWriteLock```类似，这里64位的```state```按二进制位被分为了多个部分，每个部分分别有不同的用途。其中最后（最小）7位被用来记录读锁的数量，第8位用作写锁标记为，第8位到第64位来用作戳记（包括第8位）。因为```StampedLock```是不可重入的，因此不需要记录写锁的数量，因为写锁只会有一个并且重入次数为1，所以使用1个二进制位做一下标记即可。当读计数溢出时，会使用```readerOverflow```来记录。

2. ```StampedLock```定义了多个变量方便按位操作。如下所示：

   | 属性       | 描述                                                         |
   | ---------- | ------------------------------------------------------------ |
   | LG_READERS | 值为7，是溢出前用于记录读锁计数的二进制位数                  |
| RUNIT      | ReadUnit，值为1L，跟```ReentrantReadWriteLock中```SHARED_UNIT一样，是用于给读计数加1的。 |
|WBIT|WriteBit，值为1L << LG_READERS，也就是第8位的值为1，用做写锁标记。 0000 ... 1000 0000|
   |RBITS|ReadBits，值为WBIT - 1L，二进制为低7位全部为1，其余为0。                      0000 ... 0111 1111|
   |RFULL|ReadFull，值为RBITS - 1L，表示溢出前state中读计数的最大值，十进制值为126|
   |ABITS|值为RBITS \| WBIT，用做掩码，stamp & ABITS可以得到后8位的值                 0000 ... 1111 1111|
   |SBITS|StampedBits，值为~RBITS，用做掩码，state & SBITS可以得到戳记部分       1111 ... 1000 0000|
   |ORIGIN|戳的初值，值为WBIT << 1，目的是跳过戳记的0值                                              0000 ... 0001 0000 0000|
   
3. ```StampedLock```是基于CLH队列的，内部类```WNode```用来封装等待的线程，内部属性mode区分是读还是写模式，status区分节点的状态0/WAITING/CANCELLED。

下面我们分别从写锁、悲观读锁、乐观读锁三方面来看源码。

### 写锁源码

写锁的顶级入口为```writeLock()、tryWriteLock()、tryWriteLock(long time, TimeUnit unit)```，其中```tryWriteLock()```系列跟其它锁实现一样，判断完锁状态后如果允许获得锁就尝试CAS的获取锁，如果最终获取失败（有可能是超时），直接返回而不会排队。而```writeLock()```失败后会进入同步队列排队，排队的代码是最复杂的一部分，所以我们从```writeLock()```的源码看起，看完了```writeLock()```的源码后，其余try系列的方法将会非常简单。

```java
public long writeLock() {
    long s, next;  // bypass acquireWrite in fully unlocked case only
    return ((((s = state) & ABITS) == 0L &&
             U.compareAndSwapLong(this, STATE, s, next = s + WBIT)) ?
            next : acquireWrite(false, 0L));
}
```

### 悲观读锁源码

### 乐观读锁源码



