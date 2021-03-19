[toc]

# Semaphore源码分析

> *本文源码基于JDK8*。因为本人水平有限，错误和不足之处在所难免，欢迎指出错误和不足之处，一起进步。
>
> 阅读这篇文章最好对AQS有一定的了解。

## 1.Semaphore介绍

**Semaphore**是juc中的同步工具，使用```Semaphore```可以控制同时访问资源的线程的个数。可以理解为```Semaphore```有n个许可，只有成功获得许可的线程才能访问临界区，线程使用完后会将许可返还给```Semaphore```，从而实现同时访问资源的线程的最大个数。

## 2.Semaphore的简单使用

```Semaphore```提供了两个构造器：

1. ```public Semaphore(int permits)```
2. ```public Semaphore(int permits, boolean fair)```

其中，```permits```代表许可的总数，也就是最多允许permits个线程同时访问资源；```fair```表示是否使用公平模式，```true```表示使用公平模式，```false```表示使用非公平模式，默认为非公平模式。

```Semaphore```提供了一系列的```acquire```方法来以不同方式获取许可，提供了```release```方法来释放许可。

下面的示例设置了一个拥有三个许可的```Semaphore```：

```java
import java.util.concurrent.Semaphore;
import java.util.concurrent.TimeUnit;

public class SemaphoreDemo {
    static final Semaphore semaphore = new Semaphore(3);

    static void fun() {
        try {
            System.out.println(Thread.currentThread().getName() + "尝试获得许可，此时剩余许可数量：" + semaphore.availablePermits());
            semaphore.acquire();
            //只是为了控制台输出字体颜色容易分辨
            System.err.println(Thread.currentThread().getName() +
                    "成功获得许可，当前剩余许可个数 ：" + semaphore.availablePermits());
            TimeUnit.SECONDS.sleep(2);

        } catch (InterruptedException e) {
            e.printStackTrace();
        } finally {
            semaphore.release();
        }

    }

    public static void main(String[] args) {
        for (int i = 0; i < 10; i++) {
            new Thread(SemaphoreDemo::fun).start();
        }
    }
}
```

运行代码可以看到，最开始只有三个线程输出“成功获得许可”，然后获得许可的线程慢慢增加，最终全部获得许可。我们也可以在程序开始和结束获取当前时间来获得每个线程等待的时间。这里如果想看一下每个线程获得许可后实时的剩余许可数量，可以采用加锁操作来让输出和获得许可具有原子性，感兴趣的小伙伴可以试一下，这里就不再写了。

## 3.Semaphore源码分析

跟```ReentrantLock```相似，```Semaphore```也有公平模式和非公平模式，默认为非公平模式，可以通过构造器的参数来选择公平模式或者非公平模式。```Semaphore```内部同样有三个内部类：```Sync、NonfairSync、FairSync```，三者都是AQS的子类，其中```NonfairSync、FairSync```是```Sync```的子类，下面来看一下他们的源码。

### 3.1Sync源码

```Sync```是```AbstractQueuedSynchronizer```的子类，在这里```state```表示许可的数量。假设现在让我们用AQS实现控制同时访问资源的最大线程数，我们不难想到可以使用AQS提供的共享模式来实现。```state```我们抽象为许可，我们重写的tryAcquireShared方法只需要在获取许可成功后返回剩余许可的数量即可，这样当许可小于等于0时，线程就会进入AQS的同步队列中进行调度，等到有线程释放许可后才会获得许可然后运行。思路还是比较简单的，主要是要对AQS共享模式的流程比较熟悉才会好理解（共享模式下，```tryAcquireShare```方法返回的数值代表获取成功后资源的数量，比如为0表示获取成功，但是已经没有资源了可以获取了，返回负值则表示获取失败，需要进入AQS同步队列中调度）。下面我们来看一下```Semaphore```是怎么实现的。

```java
abstract static class Sync extends AbstractQueuedSynchronizer {
    private static final long serialVersionUID = 1192457210091910933L;

    //state抽象为许可，创建时有多少许可就把state设置为多少
    //构造函数无需CAS，简单的设置值即可
    Sync(int permits) {
        setState(permits);
    }

    //返回当前时刻可用的许可数
    //注意这个值很有可能是不准确的，因为共享模式下会有多个线程修改state的值
    //应该很好理解
    final int getPermits() {
        return getState();
    }

    //tryAcquireShared的非公平版本
    //在Sync中实现该方法而不是在NonfairSync中实现的原因与ReentrantLock一样
    //主要是为了给Semaphore.tryAcquire()方法调用（在ReentrantLock中为tryLock方法）
    //小伙伴们可以想一下如果在NonfairSync实现会怎么样
    final int nonfairTryAcquireShared(int acquires) {
        //CAS+自旋，这里是很经典的多线程下安全的修改变量值的写法。
        //因为是共享模式，所以这段代码是会被多个线程执行的
        for (;;) {
            //获得当前时刻可用的许可数
            int available = getState();
            int remaining = available - acquires;
            //小于0表示剩余许可已经不能满足线程需要的许可数量了，因此不需要也不应该执行CAS了
            //其他情况下就CAS的去修改state，保证state的值原子性的修改
            //最终返回remaining，小于0的情况表示获取失败，进入AQS同步队列调度
            //大于等于0表示获取许可成功，直接返回了就
            //这里直接用CAS来改变state的值，而不管AQS的同步队列中是否有线程在等待，所以是非公平的
            if (remaining < 0 ||
                compareAndSetState(available, remaining))
                return remaining;
        }
    }

    //释放资源的方法
    protected final boolean tryReleaseShared(int releases) {
        //同获取一样，释放也需要CAS+自旋，只不过获取时是减，这里是加罢了
        for (;;) {
            int current = getState();
            int next = current + releases;
            if (next < current) // overflow
                throw new Error("Maximum permit count exceeded");
            //最终会释放成功滴
            if (compareAndSetState(current, next))
                return true;
        }
    }

    //减少许可的总数，CAS的减少reductions个许可
    final void reducePermits(int reductions) {
        for (;;) {
            int current = getState();
            int next = current - reductions;
            if (next > current) // underflow
                throw new Error("Permit count underflow");
            if (compareAndSetState(current, next))
                return;
        }
    }

    //一次性获取所有剩余许可的方法
    final int drainPermits() {
        for (;;) {
            int current = getState();
            if (current == 0 || compareAndSetState(current, 0))
                return current;
        }
    }
}
```

```Sync```中定义了非公平的资源获取方法和通用的资源释放方法以及一次性获取所有剩余许可的方法```drainPermits```。```reducePermits```方法是debug用的，不需要特别关注。

### 3.2NonfairSync源码

```NonfairSync```的代码非常简单，只是按照AQS的使用方法重写了```tryAcquireShared```方法，但是所有逻辑都是在父类```Sync```中定义的，这里不再赘述。

```java
static final class NonfairSync extends Sync {
    private static final long serialVersionUID = -2694183684443567898L;

    NonfairSync(int permits) {
        super(permits);
    }

    //直接调用父类Sync中定义的nonfairTryAcquireShared方法
    protected int tryAcquireShared(int acquires) {
        return nonfairTryAcquireShared(acquires);
    }
}
```

### 3.3FairSync源码

```FairSync```中定义了公平的获取许可的方法```tryAcquireShared```，跟非公平的获取许可的代码几乎一模一样，区别仅在于需要先判断一下是否有线程在AQS的同步队列中等待，如果有线程在等待，那么直接返回-1，表示资源获取失败，然后进入AQS的同步队列中排队。因为AQS的同步队列是FIFO的，所以实现了公平的获取许可。

```java
static final class FairSync extends Sync {
    private static final long serialVersionUID = 2014338818796000944L;

    FairSync(int permits) {
        super(permits);
    }

    protected int tryAcquireShared(int acquires) {
        //同样是AQS+自旋
        for (;;) {
            //如果同步队列中有线程在排队，那么直接返回-1，代表资源获取失败
            //返回负数情况下线程会进入到AQS的同步队列中进行调度
            if (hasQueuedPredecessors())
                return -1;
            int available = getState();
            int remaining = available - acquires;
            if (remaining < 0 ||
                compareAndSetState(available, remaining))
                return remaining;
        }
    }
}
```

### 3.4Semaphore的其他api的实现

```java
//获取1个许可的方法，没法获取许可时会阻塞。该方法是响应中断的
//从这个方法中返回有两种情况：1.获取许可成功 2.线程中断标记被设置
public void acquire() throws InterruptedException {
    sync.acquireSharedInterruptibly(1);
}
//获取1个许可的方法，没法获取许可时会阻塞。不响应中断
public void acquireUninterruptibly() {
    sync.acquireShared(1);
}
//尝试获取1个许可的方法，返回true表示成功，false表示失败
//nonfairTryAcquireShared返回的是获取资源后state的值
//大于等于0表示这次获取是成功的，否则就是失败了
public boolean tryAcquire() {
    return sync.nonfairTryAcquireShared(1) >= 0;
}
//带有超时时间的tryAcquire版本，超过指定时间后会失败
public boolean tryAcquire(long timeout, TimeUnit unit)
    throws InterruptedException {
    return sync.tryAcquireSharedNanos(1, unit.toNanos(timeout));
}
//释放一个许可
public void release() {
    sync.releaseShared(1);
}
//获取指定个数的许可，响应中断
public void acquire(int permits) throws InterruptedException {
    if (permits < 0) throw new IllegalArgumentException();
    sync.acquireSharedInterruptibly(permits);
}
//获取指定个数的许可，不响应中断    
public void acquireUninterruptibly(int permits) {
    if (permits < 0) throw new IllegalArgumentException();
    sync.acquireShared(permits);
}
//尝试获取指定个数的许可
public boolean tryAcquire(int permits) {
    if (permits < 0) throw new IllegalArgumentException();
    return sync.nonfairTryAcquireShared(permits) >= 0;
}
//带有超时时间的尝试获取指定个数的许可
public boolean tryAcquire(int permits, long timeout, TimeUnit unit)
    throws InterruptedException {
    if (permits < 0) throw new IllegalArgumentException();
    return sync.tryAcquireSharedNanos(permits, unit.toNanos(timeout));
}
//释放指定个数的许可
public void release(int permits) {
    if (permits < 0) throw new IllegalArgumentException();
    sync.releaseShared(permits);
}
//返回当前时刻可用的许可数
public int availablePermits() {
    return sync.getPermits();
}
//一次性获取所有剩余的许可
public int drainPermits() {
    return sync.drainPermits();
}
```

总结一下，```Semaphore```的设计思路跟```ReentrantLock```是很相似的，都是借助AQS来实现同步的，都有公平模式和非公平模式。并且不难看出，想用AQS实现公平模式的话，就在获取资源时（我们自己重写的tryAcquire/tryAcquireShared方法）判断一下AQS的同步队列中是否有线程在等待，如果有线程在等待，那么直接返回失败标志（独占模式下返回false，共享模式下返回负数），让当前线程进入AQS同步队列进行同步调度，因为AQS的同步队列是FIFO的，因此保证了线程获得资源的顺序和申请顺序是一致的，而非公平模式下就不需要判断是否有线程在等待，而是直接尝试获取资源。