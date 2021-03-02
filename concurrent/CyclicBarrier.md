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

那么```CyclicBarrier```是如何实现这样的功能的呢？

## CyclicBarrier源码分析

