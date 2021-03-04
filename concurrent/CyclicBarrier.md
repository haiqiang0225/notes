[toc]

# CyclicBarrier分析

> *本文源码基于JDK8*。因为本人水平有限，错误和不足之处在所难免，欢迎指出错误和不足之处，一起进步。
> 阅读这篇文章最好对AQS有一定的了解。

## CyclicBarrier介绍

**CyclicBarrier**和juc中另一个同步工具```CountDownLatch```非常像，使用```CyclicBarrier```可以使多个线程阻塞等待，等到所有线程都准备完毕了以后再一起执行。比如玩游戏的时候，进游戏前需要所有人都加载完毕才能进入，或者等所有人都准备了之后才能开始游戏。juc中提供的```CountDownLatch```和```CyclicBarrier```都可以实现这些功能，不同之处在于```CountDownLatch```是一次性的，计数器为0后无法重置，而```CyclicBarrier```提供了```reset()```方法来恢复计数，从```CyclicBarrier```的名字就能看出来它是一个可以循环使用的“栅栏”，而```CountDownLatch```则是一个计数减少的“门”。```await()```方法就是“栅栏”或者“门”，线程执行该方法后如果不满足“开门”的条件就会被阻塞在该方法处，直到条件满足。

跟```CountDownLatch```直接借助AQS的共享模式实现不同，```CyclicBarrier```借助```ReentrantLock```来实现的（实际上还是AQS啦，毕竟```ReentrantLock```也是借助AQS实现的）。我们先看段代码了解了```CyclicBarrier```如何使用之后再去看一下它的源码。

## CyclicBarrier使用

下面的Demo初始化了十个子线程，每个子线程会睡眠一会来模拟加载时的耗时，加载完毕的线程会在栅栏(```awiat()```方法)处等待，直到所有的线程都执行到栅栏处后，栅栏会放开，所有线程就可以继续执行了。

```java
import java.util.concurrent.BrokenBarrierException;
import java.util.concurrent.CyclicBarrier;
import java.util.concurrent.ThreadLocalRandom;
import java.util.concurrent.TimeUnit;

public class CyclicBarrierDemo {

    //初始化一个计数为10的实例
    static final CyclicBarrier cyclicBarrier = new CyclicBarrier(10);

    static void loading() {
        try {
            System.out.println(Thread.currentThread().getName() + " 开始加载");
            //将当前线程挂起一段时间，模拟加载时的耗时
            TimeUnit.SECONDS.sleep(ThreadLocalRandom.current().nextInt(3));
            //加载完毕的线程会在await()这里阻塞
            cyclicBarrier.await();
            System.out.println(Thread.currentThread().getName() + "加载完毕");
        } catch (InterruptedException | BrokenBarrierException e) {
            e.printStackTrace();
        }
    }

    public static void main(String[] args) throws InterruptedException {
        Thread[] threads = new Thread[10];
        //我们模拟十个子线程来加载
        for (int i = 0; i < 10; i++) {
            threads[i] = new Thread(CyclicBarrierDemo::loading);
            threads[i].start();
        }
        for (int i = 0; i < 10; i++) {
            threads[i].join();
        }
        System.out.println("-----所有子线程加载完毕-----");
        //重置栅栏，下面接着使用该栅栏
        cyclicBarrier.reset();
        for (int i = 0; i < 10; i++) {
            threads[i] = new Thread(CyclicBarrierDemo::loading);
            threads[i].start();
        }
    }
}
```

执行上面代码可以看到，“加载完毕”这条输出永远是在所有的“开始加载”输出后开始输出的。这也就说明了只有当所有方法都执行到``` cyclicBarrier.await();```这里的时候，栅栏才会被打开。

除了上面的使用方式，```CyclicBarrier```还可以定义每此屏障被突破后的动作，下面的代码展示了这一点：

## //TO-DO添加一个带barrierCommand的例子

那么```CyclicBarrier```是如何实现这样的功能的呢？最开始我是猜测和```CountDownLatch```一样是直接用AQS的共享模式实现的，```reset()```方法把AQS的```state```变量从0恢复到原来的值，当```state```不为0时```tryAcquireShared```方法返回负数，这样线程就会进入AQS同步队列等待。后来试着写了写，发现直接像```CountDownLatch```一样用AQS的共享模式的话并不是很容易实现，然后就去看了看```CyclicBarrier```是怎么实现上面的功能的。

## CyclicBarrier源码分析

在看源码之前先看一下几个成员变量

| 名称 | 解释                |
| ---- | ------------------- |
| lock | ReentrantLock的实例。 |
|trip|AQS内部类ConditionObject的实例，这里配合lock一起实现等待/通知机制。|
|parties|栅栏每代能拦截的线程总数，和CountDownLatch中的count作用相似|
|barrierCommand|构造器可选参数。是每次屏障被突破后执行的命令，传入的命令必须实现了Runnable接口|
|generation|代表当前栅栏的代数，每次栅栏被突破后都会进入到下一代，该引用指向的实例也会改变，以此来区分代数|
|count|当前剩余未到达栅栏处的线程数，也就是说再有count个线程执行await()方法，栅栏就会被突破|

其中```Generation```是内部类：

```
private static class Generation {
    boolean broken = false;
}
```

我们还是从顶层的```await()```方法的源码看起，代码如下：

```java
public int await() throws InterruptedException, BrokenBarrierException {
    try {
        return dowait(false, 0L);
    } catch (TimeoutException toe) {
        throw new Error(toe); // cannot happen
    }
}
```

可以看到代码很简单，逻辑都在```dowait(false, 0L)```这里，源码：

```java
private int dowait(boolean timed, long nanos)
    throws InterruptedException, BrokenBarrierException,
           TimeoutException {
    final ReentrantLock lock = this.lock;
    lock.lock();
    try {
        final Generation g = generation;

        if (g.broken)
            throw new BrokenBarrierException();

        if (Thread.interrupted()) {
            breakBarrier();
            throw new InterruptedException();
        }

        int index = --count;
        if (index == 0) {  // tripped
            boolean ranAction = false;
            try {
                final Runnable command = barrierCommand;
                if (command != null)
                    command.run();
                ranAction = true;
                nextGeneration();
                return 0;
            } finally {
                if (!ranAction)
                    breakBarrier();
            }
        }

        // loop until tripped, broken, interrupted, or timed out
        for (;;) {
            try {
                if (!timed)
                    trip.await();
                else if (nanos > 0L)
                    nanos = trip.awaitNanos(nanos);
            } catch (InterruptedException ie) {
                if (g == generation && ! g.broken) {
                    breakBarrier();
                    throw ie;
                } else {
                    // We're about to finish waiting even if we had not
                    // been interrupted, so this interrupt is deemed to
                    // "belong" to subsequent execution.
                    Thread.currentThread().interrupt();
                }
            }

            if (g.broken)
                throw new BrokenBarrierException();

            if (g != generation)
                return index;

            if (timed && nanos <= 0L) {
                breakBarrier();
                throw new TimeoutException();
            }
        }
    } finally {
        lock.unlock();
    }
}
```