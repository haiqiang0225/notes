[toc]

# Semaphore源码分析

> *本文源码基于JDK8*。因为本人水平有限，错误和不足之处在所难免，欢迎指出错误和不足之处，一起进步。

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

```java
abstract static class Sync extends AbstractQueuedSynchronizer {
    private static final long serialVersionUID = 1192457210091910933L;

    Sync(int permits) {
        setState(permits);
    }

    final int getPermits() {
        return getState();
    }

    final int nonfairTryAcquireShared(int acquires) {
        for (;;) {
            int available = getState();
            int remaining = available - acquires;
            if (remaining < 0 ||
                compareAndSetState(available, remaining))
                return remaining;
        }
    }

    protected final boolean tryReleaseShared(int releases) {
        for (;;) {
            int current = getState();
            int next = current + releases;
            if (next < current) // overflow
                throw new Error("Maximum permit count exceeded");
            if (compareAndSetState(current, next))
                return true;
        }
    }

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

    final int drainPermits() {
        for (;;) {
            int current = getState();
            if (current == 0 || compareAndSetState(current, 0))
                return current;
        }
    }
}
```

### 3.2NonfairSync源码

### 3.3FairSync源码