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

// 当线程获取锁失败时，会先自旋一定的次数，SPINS是进入同步队列等待前自旋的次数
/** Maximum number of retries before enqueuing on acquisition */
private static final int SPINS = (NCPU > 1) ? 1 << 6 : 0;

// 如果前驱是头结点，那么自旋次数就初始化为HEAD_SPINS，这发生在结点入队之后
/** Maximum number of retries before blocking at head on acquisition */
private static final int HEAD_SPINS = (NCPU > 1) ? 1 << 10 : 0;

// 重新阻塞前最大的尝试次数
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
    // 读模式使用该引用链接读结点
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
   | WBIT       | WriteBit，值为1L << LG_READERS，也就是第8位的值为1，用做写锁标记。                    0000 ... 1000 0000 |
   | RBITS      | ReadBits，值为WBIT - 1L，二进制为低7位全部为1，其余为0。                                         0000 ... 0111 1111 |
   | RFULL      | ReadFull，值为RBITS - 1L，表示溢出前state中读计数的最大值，十进制值为126 |
   | ABITS      | AccessBits值为RBITS \| WBIT，用做掩码，stamp & ABITS可以得到后8位的值                 0000 ... 1111 1111 |
   | SBITS      | StampedBits，值为~RBITS，用做掩码，state & SBITS可以得到戳记部分                         1111 ... 1000 0000 |
   | ORIGIN     | 戳的初值，值为WBIT << 1，目的是跳过戳记的0值                                                                0000 ... 0001 0000 0000 |

3. ```StampedLock```是基于CLH队列的，内部类```WNode```用来封装等待的线程，内部属性mode区分是读还是写模式，status区分结点的状态0/WAITING/CANCELLED。

下面我们分别从写锁、悲观读锁、乐观读锁三方面来看源码。

### 写锁源码

#### 获取写锁源码

写锁的顶级入口为```writeLock()、tryWriteLock()、tryWriteLock(long time, TimeUnit unit)```，其中```tryWriteLock()```系列跟其它锁实现一样，判断完锁状态后如果允许获得锁就尝试CAS的获取锁，如果最终获取失败（有可能是超时），直接返回而不会排队。而```writeLock()```失败后会进入同步队列排队，排队的代码是最复杂的一部分，所以我们从```writeLock()```的源码看起，看完了```writeLock()```的源码后，其余try系列的方法将会非常简单。

```java
public long writeLock() {
    long s, next;  // bypass acquireWrite in fully unlocked case only
    // ((s = state) & ABITS) == 0L 成立的话，说明当前是没有任何锁的（读锁/写锁）
    return ((((s = state) & ABITS) == 0L &&  // ↓ 没有锁的情况才会尝试CAS的去获取写锁，操作很简单
             U.compareAndSwapLong(this, STATE, s, next = s + WBIT)) ? // 就是把写锁那一位设置为1 ：next = s + WBIT
            next : acquireWrite(false, 0L)); //CAS成功返回s + WBIT，失败进入acquireWrite逻辑
}
```

从```writeLock()```的源码我们可以看出```StampedLock```的实现是非公平锁。

```acquireWrite(boolean interruptible, long deadline)```的代码非常长，下面我们配合着注释一点一点的来看

```java
private long acquireWrite(boolean interruptible, long deadline) {
    // for循环1
    WNode node = null, p;
    for (int spins = -1;;) { // spin while enqueuing
        // m是state后8位的值，s是state当前时刻的值，ns=next state
        long m, s, ns;
        // m == 0L 成立说明当前没有任何锁，尝试CAS的获取锁，成功则返回
        if ((m = (s = state) & ABITS) == 0L) {
            if (U.compareAndSwapLong(this, STATE, s, ns = s + WBIT))
                return ns;
        }
        // spins < 0 成立说明是第一次进入循环，这里初始化自旋次数
        else if (spins < 0)
            // 如果有写锁，并且没有很多线程在排队就设置为SPINS，否者自旋次数置为0
            // wtail == whead 有俩种可能，一种是两个都是null，另一个是两个都是读节点
            spins = (m == WBIT && wtail == whead) ? SPINS : 0;
        // 自旋次数大于0的话，有约50%的概率减少一次自旋
        // 也就是说大约自旋 2 * spins次，只要spins大于0，就不会入队，而是尝试CAS
        else if (spins > 0) {
            // LockSupport.nextSecondarySeed()我测试了一下，我执行1亿次该方法
            // 得到的结果是大约有一半的几率返回大于等于0的数，
            if (LockSupport.nextSecondarySeed() >= 0)
                --spins;
        }
        // 运行到此处，说明自旋已经结束，要进入入队的流程了
        // 如果CLH队列为空，就初始化CLH队列
        else if ((p = wtail) == null) { // initialize queue
            // 新建一个结点，并用CAS的方式把whead指向这个新建结点
            WNode hd = new WNode(WMODE, null);
            if (U.compareAndSwapObject(this, WHEAD, null, hd))
                wtail = hd;
        }
        // 如果node为null，新建一个写模式的WNode。p是上面读取时刻的wtail
        else if (node == null)
            node = new WNode(WMODE, p);
        // 如果刚才新建的node的pre已经不是最新的wtail了，就更新这个引用
        else if (node.prev != p)
            node.prev = p;
        // CAS的把自己设为wtail，保证队列是一致的
        else if (U.compareAndSwapObject(this, WTAIL, p, node)) {
            // 入队成功的线程把前驱的next指针指向自己
            p.next = node;
            // 入队成功后就中断这个循环
            break;
        }
    }

    // for循环2
    for (int spins = -1;;) {
        // h是whead， np = node.prev， pp = p.prev（前驱的前驱）， ps = p.status
        WNode h, np, pp; int ps;
        // p是上面成功入队的结点的前驱，node是上面成功入队的结点
        // 这个if是如果成功入队的结点的前驱已经是头结点的情况
        if ((h = whead) == p) {
            // 第一次进入循环时，初始化自旋次数为HEAD_SPINS
            if (spins < 0)
                spins = HEAD_SPINS;
            // 如果spins小于MAX_HEAD_SPINS（65536）
            else if (spins < MAX_HEAD_SPINS)
                // 相当于 spins = spins * 2
                spins <<= 1;
            for (int k = spins;;) { // spin at head
                // s ： state， ns ： next state
                long s, ns;
                // 没有任何锁的话，尝试CAS设置写标志
                if (((s = state) & ABITS) == 0L) {
                    if (U.compareAndSwapLong(this, STATE, s,
                                             ns = s + WBIT)) {
                        // 获得锁成功后，把头结点设置为自己
                        whead = node;
                        // 清空前驱的引用，帮助GC回收
                        node.prev = null;
                        return ns;
                    }
                }
                // LockSupport.nextSecondarySeed() >= 0 成立的概率接近0.5
                // --k <=0 成立的话，说明已经超过了给定的自旋次数
                else if (LockSupport.nextSecondarySeed() >= 0 &&
                         --k <= 0)
                    // 跳出这个for循环，即跳出在头结点上自旋的逻辑
                    break;
            }
        }
        // 如果成功入队的结点的前驱不是头结点且头结点不为null
        // 下面的代码是用来协助唤醒在头结点中等待的读结点的
        else if (h != null) { // help release stale waiters
            WNode c; Thread w;
            while ((c = h.cowait) != null) {
                if (U.compareAndSwapObject(h, WCOWAIT, c, c.cowait) &&
                    (w = c.thread) != null)
                    U.unpark(w);
            }
        }
        // 头结点没有改变
        if (whead == h) {
            // node.prev 跟刚才的 p不一样了，把p更新为node.prev
            // 出现这样的情况说明前驱被取消了
            if ((np = node.prev) != p) {
                if (np != null)
                    (p = np).next = node;   // stale
            }
            // ps == 0 前驱的status为0，CAS的修改为WAITING
            else if ((ps = p.status) == 0)
                U.compareAndSwapInt(p, WSTATUS, 0, WAITING);
            // 如果前驱结点取消了，就跳过这个结点
            else if (ps == CANCELLED) {
                if ((pp = p.prev) != null) {
                    node.prev = pp;
                    pp.next = node;
                }
            }
            else {
                // 阻塞的时间，这块是给带超时时间的获取方法实现的，0代表没有超时时间
                long time; // 0 argument to park means no timeout
                if (deadline == 0L)
                    time = 0L;
                // 超时后取消等待
                else if ((time = deadline - System.nanoTime()) <= 0L)
                    return cancelWaiter(node, node, false);
                Thread wt = Thread.currentThread();
                // 设置线程阻塞者PARKBLOCKER，这个对调试有用
                U.putObject(wt, PARKBLOCKER, this);
                // 要阻塞线程了，阻塞之前设置node的thread引用
                node.thread = wt;
                // 如果满足以下条件：
                //   1. p.status < 0 表示结点是有效的，大于0时表示结点被取消
                //   2. p和h不相同（前驱仍然不是头结点） 或者 有任何锁存在
                //   3. whead == h 即头结点没有发生改变 
                //   4. node的前驱为p
                // 就阻塞当前线程
                if (p.status < 0 && (p != h || (state & ABITS) != 0L) &&
                    whead == h && node.prev == p)
                    U.park(false, time);  // emulate LockSupport.park
                // 线程被唤醒后就把node.thread清空，这里应该是为了帮助GC回收线程
                node.thread = null;
                // 把PARKBLOCKER设置为null
                U.putObject(wt, PARKBLOCKER, null);
                // 如果允许中断并且已经被中断过了，就取消等待
                if (interruptible && Thread.interrupted())
                    return cancelWaiter(node, node, true);
            }
        }
    }
}
```

这段代码真的非常长啊（对不起我不该联想到老太太的裹脚布），我们还是先来梳理一下吧。  
```acquireWrite```方法包含两个for循环：注释里的for循环1和for循环2。

其中for循环1包含了以下逻辑：

1. 首先自旋尝试获得写锁
2. 自旋一定次数还没有获得写锁的话，就进入入队的逻辑
3. 成功入队后跳出这个for循环

也就是说，线程执行跳出第一个for循环后，就已经成功的进入同步队列了，之后进入第二个for循环。

第二个for循环的逻辑：

1. 如果成功入队的结点的前驱结点是头结点的话，说明当前结点很有可能会很快有机会获得锁，因此先自旋一定的次数而不是阻塞。自旋一定的次数还没有获得锁的话，停止自旋。如果入队后的自旋成功获得锁的话，线程会把自己设置为头结点，也就是说**这一点和AQS的CLH队列是一样的，头结点要么是最开始入队时初始化的一个空结点（没有线程），要么是已经获得了锁的结点。**

2. 如果成功入队的结点的前驱结点不是头结点的话，就去协助唤醒等待在头结点上的读结点（如果有的话）。

3. 除非前面1中获取锁成功返回了，否则就进入阻塞的流程，在阻塞之前先判断一下前驱结点的状态，跳过被取消的结点。满足条件的线程会执行```LockSupport.park()```方法阻塞，等待唤醒。被唤醒的线程继续从第二个for循环的开始处执行，直到最终获得锁或者被取消。

啊，终于看完获取写锁的源码了，其实这段代码逻辑还是比较明了的，不过这个代码确实太长了，在对```Stampedlock```还不是特别了解的时候看这个代码真的是一件非常折磨人的事。不过这里的代码对比后面获取悲观读锁的代码还是要简单一些的，小伙伴们保持耐心继续看吧，任重而道远啊。

看完了怎么获取写锁之后，自然是要看一下释放的逻辑了。不要担心，释放的逻辑并不复杂。

#### 释放写锁源码

写锁释放的顶层入口是```unlockWrite(long stamp)```方法，我们来看一下它的源码。

```java
public void unlockWrite(long stamp) {
    WNode h;
    // 如果当前的戳被改变，抛出异常
    // 因为我们是写锁（独占锁），理论上执行到这里时stamp不会
    // 改变，除非使用锁的人（我们）写的代码有问题[doge]
    if (state != stamp || (stamp & WBIT) == 0L)
        throw new IllegalMonitorStateException();
    // 更新state的值，如果stamp + WBIT == 0，
    // （此时的stamp应该为1111 ... 1000 0000）
    // 就跳过这个值，用ORIGIN来重新初始化state，否则就用
    // stamp + WBIT来更新state，这样第8位就会由1变为0，并且
    // 戳记的部分也会增加1。因为是独占，不需要同步
    state = (stamp += WBIT) == 0L ? ORIGIN : stamp;
    // 队列不为空并且头结点状态不是0，调用release方法
    // 在这里如果头结点状态为0表示头结点已经执行过release方法了
    // 因为release不只在该方法中被调用h.status == 0说明已经执行过release方法了
    if ((h = whead) != null && h.status != 0)
        release(h);
}

private void release(WNode h) {
    // 防止出现npe
    if (h != null) {
        WNode q; Thread w;
        // CAS的把头结点的状态由WAITING修改为0
        U.compareAndSwapInt(h, WSTATUS, WAITING, 0);
        // 如果头结点的next指针为空或者next节点被取消了
        if ((q = h.next) == null || q.status == CANCELLED) {
            // 就从队尾向前找有效结点，为什么从后往前找呢？
            // 这里和AQS是一样的，next指针仅作为一种优化，在入队时，线程执行CAS把自己设
            // 为队尾成功时，prev指针是已经被连接的，并且结点也已经入队了，但是这个时候
            // p.next = node（StampedLock）或者pred.next = node（AQS）这句语句有
            // 可能没有执行，这个时候如果用next指针遍历CLH队列的话，很有可能会出现一种情况
            // 就是遍历不到tail指向的那个结点，所以next指针应该用作启发式的作用，真要强
            // 一致性还是要看我prev指针的[doge]。
            for (WNode t = wtail; t != null && t != h; t = t.prev)
                if (t.status <= 0)
                    q = t;
        }
        // 防止npe 
        if (q != null && (w = q.thread) != null)
            // 唤醒有效后继结点的线程来获取锁
            U.unpark(w);
    }
}
```

释放的逻辑和上面获取的逻辑比起来可以说是灰常简单了，完成了两件事：

1. 释放锁，```state = (stamp += WBIT) == 0L ? ORIGIN : stamp;```

2. 唤醒有效后继结点来获取锁，```release(h)```

戳记显然经过一定的获取写锁/释放写锁会重复使用，比如当前戳记部分为全1后，下一个戳记就是最开始的只有一个1的戳记了，即```ORIGIN```。

至此，写锁获取与释放的源码分析完成。

### 悲观读锁源码

#### 获取锁源码

获取悲观读锁的顶级入口为```readLock()```，同写锁一样，这里不看try系列的源码。看完```readLock()```的源码后再去看try系列的方法将会非常简单。

```readLock()```方法会返回一个```long```型的戳记，释放悲观读锁时需要传入这个戳记。理论上这个戳记再释放锁时不会改变，并且获得戳记后如果有其它线程也获取了悲观读锁/乐观读锁，戳记并不会被改变。也就是说，只有获得写锁、释放写锁后，戳记才会改变。

```java
public long readLock() {
    long s = state, next;  // bypass acquireRead on common uncontended case
    // 1. wtail == whead 有俩种可能，一种是两个都是null，另一个是两个都是读节点
    // 2. 如果1成立，并且读线程的计数还没有溢出，就CAS的尝试获取读锁，如果CAS成功了，返回戳记
    // 3. 如果上面的情况有任何一个不满足，进入acquireRead逻辑
    return ((whead == wtail && (s & ABITS) < RFULL &&
             U.compareAndSwapLong(this, STATE, s, next = s + RUNIT)) ?
            next : acquireRead(false, 0L));
}
```

```readLock()```本身的逻辑还是很简单的，如果获取悲观读锁的条件满足，就尝试CAS的获取悲观读锁，成功就返回；而如果条件不满足或者说虽然条件满足，但是CAS操作失败了，那么就进入```acquireRead```的逻辑。

下面的代码很长，小伙伴们做好心理准备啊，看到这里已经经历了八十难了，离九九八十一难就差这一个了，看完差不多可以立地成佛了，嘿嘿嘿。

```java
/**
 * 通过形参就不难看出，超时不超时啥的逻辑都在这个方法了，所以，弄懂这个方法，别的
 * 方法源码看起来就如张飞吃豆芽--小菜一碟。
 */
private long acquireRead(boolean interruptible, long deadline) {
    WNode node = null, p;
    // 内部仍然是两个大的for循环
    for (int spins = -1;;) {
        WNode h;
        // h 是whead， p 是 wtail，也就是prev
        // 如果头结点和尾结点相同了，就先进入下面自旋的逻辑
        if ((h = whead) == (p = wtail)) {
            // 这个for循环是入队前自旋的逻辑
            for (long m, s, ns;;) {
                // m 是后8位的值，如果没有超过RFULL，就尝试CAS修改state，获取悲观读锁
                // 否则，判断有没有写锁， m < WBIT如果不成立，写锁位一定为1，没有写锁的
                // 话就尝试增加readerOverflow的值，成功返回心得stamp即ns，不为0，失败返回0L
                if ((m = (s = state) & ABITS) < RFULL ?
                    U.compareAndSwapLong(this, STATE, s, ns = s + RUNIT) :
                    (m < WBIT && (ns = tryIncReaderOverflow(s)) != 0L))
                    // 这里返回有两种情况，一种是CAS成功，另一种是增加readerOverflow成功
                    // 总之如果获取悲观读锁成功的话就直接在这里返回了
                    return ns;
                // m >= WBIT 说明有写锁，这个时候就不能获取悲观读锁了
                else if (m >= WBIT) {
                    // 自旋，慢慢减少自旋次数
                    if (spins > 0) {
                        if (LockSupport.nextSecondarySeed() >= 0)
                            --spins;
                    }
                    else {
                        // 说明已经自旋了一定次数了
                        if (spins == 0) {
                            // nh : new head  np : new prev
                            WNode nh = whead, np = wtail;
                            // 头尾结点均未发生改变 或者 虽然发生了改变，但是头结点不等于尾结点
                            // 那么就停止自旋，否则一直从上面的for循环开始处执行，尝试获得锁
                            if ((nh == h && np == p) || (h = nh) != (p = np))
                                break;
                        }
                        // 初始化自旋次数，显然只有第一次进入时会执行
                        spins = SPINS;
                    }
                }
            }
        }
        /* 自旋阶段没有能获得悲观读锁，接下来就该入队和等待唤醒的逻辑了 */
        // p 是 wtail，如果队列为空的话，就新建一个WMODE的结点作为头结点
        if (p == null) { // initialize queue
            WNode hd = new WNode(WMODE, null);
            if (U.compareAndSwapObject(this, WHEAD, null, hd))
                wtail = hd;
        }
        // node还没创建的话，就创建一个node
        else if (node == null)
            node = new WNode(RMODE, p);
        // 头结点等于尾结点 或者 尾结点不是读模式结点
        // 也就是说，前驱如果是读模式的话，当前结点是永远都不会
        // 将自己入队的，而是一直挂在前驱的cowait上
        else if (h == p || p.mode != RMODE) {
            // 如果尾结点发生了改变
            if (node.prev != p)
                // 指向新的尾结点
                node.prev = p;
            // 尾结点没改变的话，CAS的将尾结点指向自己，尝试入队
            else if (U.compareAndSwapObject(this, WTAIL, p, node)) {
                p.next = node;
                // 入队成功就跳出一个大循环
                break;
            }
        }
        // 如果尾结点是读模式，尝试将当前结点入栈。操作方法就是把尾结点的cowait指向自己，
        // 并且把自己的cowait指向尾结点原来的cowait，最终就成了一条链，链接尾结点上
        // 成功入栈的话，这个if判定失败，进入下面的else
        else if (!U.compareAndSwapObject(p, WCOWAIT,
                                         node.cowait = p.cowait, node))
            // 失败的话，cowait重新设置为null
            node.cowait = null;
        else {
            /**
             * 进入这个自旋，说明上面的if全部判断失败，梳理下：
             *  1. 队列不为空
             *  2. node不为空（已创建当前线程对应的node实例）
             *  3. 头尾节点没有发生变化，并且尾结点不为读模式
             *  4. 尾结点虽然是读模式，并且CAS的修改cowait成功
             * 并且结点还没能成功入队
             */
            for (;;) {
                // 显然，pp = p.prev; c = h.cowait、 w = head.thread
                WNode pp, c; Thread w;
                // 头结点不为null，并且头结点上cowait不为null，协助唤醒在其上等待的读结点
                if ((h = whead) != null && (c = h.cowait) != null &&
                    U.compareAndSwapObject(h, WCOWAIT, c, c.cowait) &&
                    (w = c.thread) != null) // help release
                    U.unpark(w);
                // 前驱结点的前驱为头结点，或者前驱结点就是头结点，或者前驱的前驱为null
                // 上面几种情况都表明了一点，就是CLH队列里已经没有多少结点在等待了
                if (h == (pp = p.prev) || h == p || pp == null) {
                    // m = 后8位的值， s = state， ns = next state
                    long m, s, ns;
                    do {
                        // 仍然是自旋获得悲观读锁的逻辑，当有写锁时，会跳出这段逻辑
                        if ((m = (s = state) & ABITS) < RFULL ?
                            U.compareAndSwapLong(this, STATE, s,
                                                 ns = s + RUNIT) :
                            (m < WBIT &&
                             (ns = tryIncReaderOverflow(s)) != 0L))
                            return ns;
                    } while (m < WBIT);
                }
                // 头结点没有变，并且前驱的前驱没有被取消
                if (whead == h && p.prev == pp) {
                    long time;
                    // 前驱的前驱为空，或者前驱为头结点，或者前驱已经被取消
                    // 那么就跳出这段for循环，回到最初那个最大的for循环开始执行
                    if (pp == null || h == p || p.status > 0) {
                        node = null; // throw away
                        break;
                    }
                    // 如果没有定时
                    if (deadline == 0L)
                        time = 0L;
                    // 超时就取消
                    else if ((time = deadline - System.nanoTime()) <= 0L)
                        return cancelWaiter(node, p, false);
                    Thread wt = Thread.currentThread();
                    // 看到这里很明显了，终于要阻塞当前线程了
                    U.putObject(wt, PARKBLOCKER, this);
                    node.thread = wt;
                    // 这个判断就是说明，当前线程真的不能马上获得锁
                    if ((h != pp || (state & ABITS) == WBIT) &&
                        whead == h && p.prev == pp)
                        U.park(false, time);
                    // 被唤醒了，又要进入下一轮获取了
                    node.thread = null;
                    U.putObject(wt, PARKBLOCKER, null);
                    if (interruptible && Thread.interrupted())
                        return cancelWaiter(node, p, true);
                }
            }
        }
    }

    /* 第二个大的自旋 */
    // 进入这个自旋只有一种可能，就是上面node执行cas(this, WTAIL, p, node)
    // 成功，说明当前结点成功进入CLH队列，入队后会break跳出最外层大的for循环,
    // 并且从上面入队的条件可以发现，入队的结点的前驱结点一定不是读模式的结点,
    // 也可以看出，如果有多个读线程同时入队的话，这些个线程会链接在第一个成功入队
    // 的读结点的cowait上，形成一个链（也可以说是栈，符合先入后出）。
    for (int spins = -1;;) {
        // 老几样， h = head， np = node.prev pp = pred.prev ps = pred.status
        WNode h, np, pp; int ps;
        // 前驱结点已经是头结点了
        if ((h = whead) == p) {
            // 那么就自旋，期望获得读锁
            if (spins < 0)
                spins = HEAD_SPINS;
            // 多自旋几次
            else if (spins < MAX_HEAD_SPINS)
                spins <<= 1;
            // 在头部自旋的逻辑，不用看也知道又是获得悲观读锁的逻辑
            for (int k = spins;;) { // spin at head
                // 老三样 m 是后8位的值， s = state， ns = next state
                long m, s, ns;
                // 条件满足就尝试获得锁
                if ((m = (s = state) & ABITS) < RFULL ?
                    U.compareAndSwapLong(this, STATE, s, ns = s + RUNIT) :
                    (m < WBIT && (ns = tryIncReaderOverflow(s)) != 0L)) {
                    // 成功获得锁了，就唤醒链接在自己身上的兄弟们，一起来获得悲观读锁
                    WNode c; Thread w;
                    whead = node;
                    node.prev = null;
                    // 唤醒兄弟们的逻辑
                    while ((c = node.cowait) != null) {
                        if (U.compareAndSwapObject(node, WCOWAIT,
                                                   c, c.cowait) &&
                            (w = c.thread) != null)
                            U.unpark(w);
                    }
                    // 兄弟们唤醒完了，收工！
                    return ns;
                } // 如果不满足获得锁的条件，并且自旋已经到了一定的次数了，就跳出在头部自旋的逻辑
                else if (m >= WBIT &&
                         LockSupport.nextSecondarySeed() >= 0 && --k <= 0)
                    break;
            }
        }
        // 前驱不是头结点，并且头结点不为null
        else if (h != null) {
            WNode c; Thread w;
            // 协助唤醒头结点上的读线程（如果有的话）
            while ((c = h.cowait) != null) {
                if (U.compareAndSwapObject(h, WCOWAIT, c, c.cowait) &&
                    (w = c.thread) != null)
                    U.unpark(w);
            }
        }
        /* 前面的自旋获取锁又双叒叕失败了（咋那么不争气那）*/
        // 如果头结点没有发生改变，发生改变的话又要跳到前面自旋的逻辑去了
        // 因为有可能又有机会获得锁了
        if (whead == h) {
            // 如果前面有失效的结点，跳过这些节点，然后回到自旋逻辑
            // 又有可能很快可以获得锁了
            if ((np = node.prev) != p) {
                if (np != null)
                    (p = np).next = node;   // stale
            }
            // 如果前驱的status为0，就尝试设为WATING，然后回到前面自旋
            else if ((ps = p.status) == 0)
                U.compareAndSwapInt(p, WSTATUS, 0, WAITING);
            // 如果前驱被取消了，跳过前驱，然后继续自旋
            else if (ps == CANCELLED) {
                if ((pp = p.prev) != null) {
                    node.prev = pp;
                    pp.next = node;
                }
            }
            // 很不幸，上面的好事都没有发生，又要阻塞了
            else {
                long time;
                if (deadline == 0L)
                    time = 0L;
                // 超时取消
                else if ((time = deadline - System.nanoTime()) <= 0L)
                    return cancelWaiter(node, node, false);
                Thread wt = Thread.currentThread();
                // 阻塞
                U.putObject(wt, PARKBLOCKER, this);
                node.thread = wt;
                if (p.status < 0 &&
                    (p != h || (state & ABITS) == WBIT) &&
                    whead == h && node.prev == p)
                    U.park(false, time);
                node.thread = null;
                U.putObject(wt, PARKBLOCKER, null);
                // 如果是响应中断的模式，并且被中断了，那么取消调度
                if (interruptible && Thread.interrupted())
                    return cancelWaiter(node, node, true);
            }
        }
    }
}
```

终于看完了，泪目。梳理一下```acquireRead```的流程，按两个大的for循环来看。

第一个大的for循环：

1. 如果头结点和尾结点相同，就先自旋一定的次数，尝试去获得悲观读锁，成功后返回戳记，失败后尝试入队
2. 如果是第一个入队的，那么就先初始化队列，初始化完成后，从1重新开始（注意初始化的第一个节点为WMODE）
3. 队列不为空，但是当前线程对应的结点还没有创建，那么就去创建，创建完成后，再次从1开始
4. 当前结点没法获得锁（123均判定失败），如果头结点等于尾结点或者虽然不相同，但是尾结点是读模式结点，则准备入队，分以下两种情况
   1. 如果尾结点发生了改变，就更新自己的prev指向新的尾结点，然后回到1
   2. 尾结点没改变的话，当前线程就会尝试CAS的更新wtail来入队，入队成功后跳出第一个大循环
5. 如果尾结点是读模式的话，尝试把自己链到尾结点的```cowait```上，成功的话，进入6，失败的话，返回1
6. 上面的情况都不满足，又进入一个自旋
   1. 协助唤醒头结点上的读结点（如果有的话）
   2. 再次判断一下是否快要获得锁了，是的话再次自旋尝试获得锁
   3. 经过一系列判断后，阻塞当前线程（到这里并没有成功入队）

第一个大的for循环的逻辑大致就是各种尝试获得悲观写锁，如果最后一个结点是读结点的话，就把自己链接到它的```cowait```上，如果自旋了一定的次数，并且头结点尾结点都没有发生改变的话，才尝试入队，成功入队后跳出第一个大的for循环，进入第二个大的for循环。

第二个大的for循环：

1. 前驱是头结点的话，各种判断是否很快/有资格获得悲观读锁，如果条件都满足的话，就自旋尝试获取悲观读锁，如果成功获得锁，就唤醒“挂”在自己身上的兄弟们，让它们也来尝试获得锁（注意这个时候当前线程已经作为结点进入同步队列了）如果不满足获得锁的条件，或者已经自旋了一定的次数了，跳出自旋的逻辑，进入3

2. 前驱不是头结点的话，协助唤醒头结点上的线程，然后进入3

3. 还是先判断CLH队列是否发生变化等等能让自己有机会获得锁的事情有没有发生（贼心不死啊），没有发生的话就阻塞，发生了的话，再次自旋。阻塞后被唤醒就再次进入到第二个for循环开始处执行

看完这里的代码我们大致知道这里实现的CLH队列是什么样的了，**首先最先初始化的头结点是一个```WMODE```的空结点，之后随着线程释放锁获取锁的操作，如果有结点成功获得读锁，那么这个结点就会成为头结点，之后的获取读和写锁的方法都会协助唤醒这个结点上链接的读结点，如果这些读结点又获取锁失败了（争点气啊），那么这些结点自旋一阵后，又会进入同步队列排队，第一个读结点入队时会插入队列中，之后的读结点会链接在这个结点的```cowait```上**。文章的最后有CLH队列的结构图，小伙伴们可以跳到最后看一看。

看完这部分源码不禁感叹：一个小小的线程获取个锁都要历经九九八十一难，泪目。

#### 释放锁源码

```unlockRead(long stamp)```是释放悲观读锁的顶级入口，这里的源码就比较简单了

```java
public void unlockRead(long stamp) {
    long s, m; WNode h;
    for (;;) {
        // 如果戳记改变了，或者读线程计数为0，或者有读锁，抛出异常
        if (((s = state) & SBITS) != (stamp & SBITS) ||
            (stamp & ABITS) == 0L || (m = s & ABITS) == 0L || m == WBIT)
            throw new IllegalMonitorStateException();
        // 如果没有溢出，就从state读计数部分减少计数值
        if (m < RFULL) {
            if (U.compareAndSwapLong(this, STATE, s, s - RUNIT)) {
                // RUNIT = 1，m == RUNIT说明读锁已经全部释放，如果头结点不为null
                // 且status不为0的话就执行release方法唤醒后继
                if (m == RUNIT && (h = whead) != null && h.status != 0)
                    release(h);
                break;
            }
        }
        // 否则减少溢出部分的计数值
        else if (tryDecReaderOverflow(s) != 0L)
            break;
    }
}
// 唤醒后继结点的方法，跟写锁释放中的是同一个方法
private void release(WNode h) {
    if (h != null) {
        WNode q; Thread w;
        U.compareAndSwapInt(h, WSTATUS, WAITING, 0);
        if ((q = h.next) == null || q.status == CANCELLED) {
            for (WNode t = wtail; t != null && t != h; t = t.prev)
                if (t.status <= 0)
                    q = t;
        }
        if (q != null && (w = q.thread) != null)
            U.unpark(w);
    }
}
```

### 乐观读锁源码

在看完写锁和悲观读锁的源码后，根据乐观锁的使用方法，想必聪明的你已经大概想到了乐观读锁是怎么实现的了，每次调用乐观读方法应该是返回当前时刻的戳记，之后执行的```validate(long)```方法应该是把这个传入的戳记和内部```state```中的戳记比较一下，相同说明没有写入，不相同说明发生了写入，下面看一下源码

```java
public long tryOptimisticRead() {
    long s;
    // 先判断有没有写锁，
    return (((s = state) & WBIT) == 0L) ? (s & SBITS) : 0L;
}

public boolean validate(long stamp) {
    // 读内存屏障，禁止load指令被重排序穿过屏障，即不允许屏障前的load
    // 指令被重排序到屏障之后，也不允许屏障后的load指令被重排序到屏障之前
    U.loadFence();
    // 如果戳记改变，返回false，否则返回true
    return (stamp & SBITS) == (state & SBITS);
}
```

至此，```Stampedlock```主要的源码都分析完成。```Stampedlock```中的CLH队列结构如下：

### 内部CLH队列结构

![CLH队列](https://img-blog.csdnimg.cn/20210317101523165.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzE5MDA3MzM1,size_16,color_FFFFFF,t_70#pic_center)


我们不妨写个程序验证一下。下面的代码先创建一个线程占有写锁，然后创建5个读线程获取悲观读锁，之后创建一个线程尝试获得写锁，我们在对应位置打上断点。

![调试](https://img-blog.csdnimg.cn/20210317101540180.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzE5MDA3MzM1,size_16,color_FFFFFF,t_70#pic_center)


运行代码，先暂停在最后获得写锁的地方。

![调试](https://img-blog.csdnimg.cn/20210317101557637.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzE5MDA3MzM1,size_16,color_FFFFFF,t_70#pic_center)


可以看到，这时已经建立了头结点，并且有读结点在其后等待，之后我们再继续调试，进入```writeLock()```方法中，执行到该结点也入队。

![调试](https://img-blog.csdnimg.cn/20210317101613363.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzE5MDA3MzM1,size_16,color_FFFFFF,t_70#pic_center)


可以看到此时队列中有一个空头结点和一个读模式结点，和最后处于写模式的尾结点。小伙伴们也可以自己写demo来验证，这里就到此为止吧。