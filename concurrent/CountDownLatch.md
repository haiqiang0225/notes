[toc]

# CountDownLatch源码分析

> *本文源码基于JDK8*。因为本人水平有限，错误和不足之处在所难免，欢迎指出错误和不足之处，一起进步。
> 阅读这篇文章最好对AQS有一定的了解。

## 1.CountDownLatch介绍

**CountDownLatch**是juc中提供的同步工具。```CountDownLatch```就像是一扇门，在门没有打开之前所有线程都要在门前面等待，只有门打开了之后才能通过。换成人话就是，```CountDownLatch```可以控制某（几）个线程等待，直到```CountDownLatch```计数为0。下面看个使用的例子就一目了然了。

## 2.CountDownLatch使用

下面的例子模拟了十个子线程加载某些资源，主线程等待子线程加载，只有子线程全部加载完毕后，主线程才可以继续运行。

```java
import java.util.concurrent.CountDownLatch;
import java.util.concurrent.ThreadLocalRandom;
import java.util.concurrent.TimeUnit;

public class CountDownLatchDemo {

    static final CountDownLatch countDownLatch = new CountDownLatch(10);

    //加载方法
    static void loading() {
        ThreadLocalRandom random = ThreadLocalRandom.current();
        int rate = 0;
        try {
            //模拟加载过程
            while (rate < 100) {
                int next = random.nextInt(50);
                if (rate + next > 100) {
                    rate = 100;
                    System.err.println(Thread.currentThread().getName() + "已加载：100%");
                    System.err.flush();
                    break;
                }
                System.out.println(Thread.currentThread().getName() + "已加载：" + rate + "%");
                rate += next;
                TimeUnit.SECONDS.sleep(random.nextInt(5));
            }
        } catch (InterruptedException e) {
            e.printStackTrace();
        } finally {
            countDownLatch.countDown();
        }
    }

    public static void main(String[] args) {
        for (int i = 0; i < 10; i++) {
            new Thread(CountDownLatchDemo::loading).start();
        }
        try {
            //主线程在这里阻塞，直到所有子线程都运行结束
            countDownLatch.await();
            System.out.println("所有线程加载完毕，主线程继续运行");
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
}
```
运行该代码可以看到，```System.out.println("所有线程加载完毕，主线程继续运行");```会在子线程运行结束后输出。可以看到，```CountDownLatch```和它的名字一样，内部维护着一个我们给出的计数，然后在满足条件后执行```CountDownLatch#countDown()```来减少一个计数，当计数值为0时，所有阻塞在```CountDownLatch#await()```方法上的线程会被唤醒并且继续运行。那么```CountDownLatch```是如何实现这个功能的呢？我们直接看源码，通过源码来了解是如何实现该功能的。
## 3.CountDownLatch源码

### 3.1内部类Sync

显然内部类```Sync```是AQS的子类。AQS是juc中定义的同步框架，内部维护了```state```变量，在```CountDownLatch```中抽象为计数的个数，并且AQS中定义了获取资源失败后调度的方法。在使用AQS构建共享模式的同步工具时，需要重写```tryAcquireShare/tryReleaseShared```，其中```tryAcquireShared```返回值为获取资源后仍剩余的资源个数，小于0表示没有资源可以获取。那么让我们来使用AQS构建```CountDownLatch```该如何构建呢？我们知道，获取资源失败的线程会进入AQS的同步队列中调度，在这里获取失败就是```tryAcquireShared```方法返回负值，那么我们就可以利用这一点来构建```CountDownLatch```。当```state```不为0时，执行```tryAcquireShared```就返回负数，这时候线程就会进入AQS的同步队列进行调度，当```state```为0之后就返回正数，同步队列中的线程就会逐个唤醒。下面看一下```CountDownLatch```中是如何实现的。

```java
private static final class Sync extends AbstractQueuedSynchronizer {
    private static final long serialVersionUID = 4982264981922014374L;
    //计数值
    Sync(int count) {
        //设置state为计数值
        setState(count);
    }

    int getCount() {
        return getState();
    }
    //和我们想的一样，state不为0时返回-1，表示资源获取失败,进入AQS同步队列进行调度
    //为0时返回1，表示资源获取成功，当state为0之后，所有在AQS同步队列中
    //等待的线程会逐个被唤醒
    protected int tryAcquireShared(int acquires) {
        return (getState() == 0) ? 1 : -1;
    }

    //释放资源的方法
    //内部就是经典的CAS+自旋，原子性的将state减一
    //也就是说，一个countDown方法是与该方法对应的
    protected boolean tryReleaseShared(int releases) {
        // Decrement count; signal when transition to zero
        for (;;) {
            int c = getState();
            if (c == 0)
                return false;
            int nextc = c-1;
            if (compareAndSetState(c, nextc))
                //我们知道，在共享模式下，当线程执行tryReleaseShared返回true时
                //会唤醒后继的线程来获取资源，所以这里只有当state为0之后才返回true
                //防止无效的唤醒
                return nextc == 0;
        }
    }
}
```

注释里写的都很清楚了，这里不再赘述了。到这里其实就已经实现了```CountDownLatch了```，下面看看暴露给我们的api是如何实现的。

```java
public void await() throws InterruptedException {
    sync.acquireSharedInterruptibly(1);
}

public boolean await(long timeout, TimeUnit unit)
    throws InterruptedException {
    return sync.tryAcquireSharedNanos(1, unit.toNanos(timeout));
}

public void countDown() {
    sync.releaseShared(1);
}
```

实现也很简单，因为AQS把相应的调度逻辑都实现好了，所以这里并没有做太多工作。

需要注意的是，```CountDownLatch```的实现中```state```是不能被重置的，也就是说```CountDownLatch```是一次性的，计数值只能减不能加，如果想重复使用，可以使用juc中的另一个同步工具```CyclicBarrier```。