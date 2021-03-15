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
    //TO-DO 读模式使用该引用
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
   
3. ```StampedLock```是基于CLH队列的，内部类```WNode```用来封装等待的线程，内部属性mode区分是读还是写模式，status区分结点的状态0/WAITING/CANCELLED。

下面我们分别从写锁、悲观读锁、乐观读锁三方面来看源码。

### 写锁源码

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

//TO-DO ： 从```writeLock()```的源码我们可以看出```StampedLock```的实现是非公平锁

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
            spins = (m == WBIT && wtail == whead) ? SPINS : 0;
        // 自旋次数大于0的话，有约50%的概率减少一次自旋
        // 也就是说大约自旋 2 * spins次，只要spins大于0，就不会入队，而是尝试CAS
        else if (spins > 0) {
            // LockSupport.nextSecondarySeed()我测试了一下，执行1亿次
            // 大约有一半的几率返回大于等于0的数，
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
                        whead = node;
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

1. 如果成功入队的结点的前驱结点是头结点的话，说明当前结点很有可能会很快有机会获得锁，因此先自旋一定的次数而不是阻塞。自旋一定的次数还没有获得锁的话，停止自旋。

2. 如果成功入队的结点的前驱结点不是头结点的话，就去协助唤醒等待在头结点上的读结点（如果有的话）。

3. 除非前面1中获取锁成功返回了，否则就进入阻塞的流程，在阻塞之前先判断一下前驱结点的状态，跳过被取消的结点。满足条件的线程会执行```LockSupport.park()```方法阻塞，等待唤醒。被唤醒的线程继续从第二个for循环的开始处执行，直到最终获得锁或者被取消。

啊，终于看完获取写锁的源码了，其实这段代码逻辑还是比较明了的，不过这个代码确实太长了，在对```Stampedlock```还不是特别了解的时候看这个代码真的是一件非常折磨人的事。不过这里的代码对比后面获取悲观读锁的代码还是要简单一些的，小伙伴们保持耐心继续看吧，任重而道远啊。

看完了怎么获取写锁之后，自然是要看一下释放的逻辑了。不要担心，释放的逻辑并不复杂。

写锁释放的顶层入口是```unlockWrite(long stamp)```方法，我们来看一下它的源码。

```java
public void unlockWrite(long stamp) {
    WNode h;
    // 如果当前的戳被改变，抛出异常
    // 因为我们是写锁（独占锁），理论上执行到这里时stamp不会
    // 改变，除非使用锁的人（我们）写的代码有问题[doge]
    if (state != stamp || (stamp & WBIT) == 0L)
        throw new IllegalMonitorStateException();
    state = (stamp += WBIT) == 0L ? ORIGIN : stamp;
    if ((h = whead) != null && h.status != 0)
        release(h);
}
```

### 悲观读锁源码

一个小小的线程获取个锁都要历经九九八十一难，泪目。

### 乐观读锁源码



