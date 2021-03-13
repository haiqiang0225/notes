# StampedLock

## StampedLock介绍

**StampedLock**是JDK1.8中新增的同步工具，通过它的名字我们可能想到它和“戳”有关，这个后面再说。跟```ReentrantReadWriteLock```作用相似，适用于读多写少的场景。与```ReentrantReadWriteLock```的不同之处有以下几点：

1. ```StampedLock```提供了**乐观读锁、悲观读锁、写锁**三种锁，而```ReentrantReadWriteLock```只有悲观读锁和写锁两种模式。

2. ```StampedLock```是不可重入的，而```ReentrantReadWriteLock```是可重入的。

3. ```StampedLock```不是基于AQS的，```ReentrantReadWriteLock```是基于AQS的。

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
        // jdk10中提供了新的api来优化，这里下面再说
        return Math.sqrt(currentX * currentX + currentY * currentY);
    }

    void moveIfAtOrigin(double newX, double newY) { // upgrade
        // Could instead start with optimistic, not read mode
        long stamp = sl.readLock();
        try {
            while (x == 0.0 && y == 0.0) {
                long ws = sl.tryConvertToWriteLock(stamp);
                if (ws != 0L) {
                    stamp = ws;
                    x = newX;
                    y = newY;
                    break;
                }
                else {
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
// 写标记位 0000 ... 1000 0000
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
```

```java
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
transient ReadLockView readLockView;
transient WriteLockView writeLockView;
transient ReadWriteLockView readWriteLockView;

/** Lock sequence/state */
private transient volatile long state;
/** extra reader count when state read count saturated */
private transient int readerOverflow;
```