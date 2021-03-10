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

内部类```Sync```继承了```AbstractQueuedSynchronizer```，AQS的```state```变量被划分成了两部分，其中高16位表示共享锁（读锁）占有的数量，低16位表示排它锁（写锁）占有的数量。下面是```Sync```中的成员变量。

```java
//共享锁的偏移数，Java里int为32位，这里把int型的state分为了高16位和低16位两部分
static final int SHARED_SHIFT   = 16;
//二进制：00000000 00000001 00000000 00000000
static final int SHARED_UNIT    = (1 << SHARED_SHIFT);
//最大值显然是16位全1，从上面SHARED_UNIT的二进制不难看出，SHARED_UNIT-1就是最大值
static final int MAX_COUNT      = (1 << SHARED_SHIFT) - 1;
//用来从state中取出独占锁的占有数量
//二进制：00000000 00000000 11111111 11111111
//显然只要用state与EXCLUSIVE_MASK执行按位与操作就能得到独占锁的占有数量
static final int EXCLUSIVE_MASK = (1 << SHARED_SHIFT) - 1;

//下面两个Api分别用来获得共享锁和独占锁的数量
/** Returns the number of shared holds represented in count  */
static int sharedCount(int c)    { return c >>> SHARED_SHIFT; }
/** Returns the number of exclusive holds represented in count  */
static int exclusiveCount(int c) { return c & EXCLUSIVE_MASK; }
```

