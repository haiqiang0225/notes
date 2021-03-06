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

除了上面的使用方式，```CyclicBarrier```还可以定义每此屏障被突破后的动作，把下面的例子中构造```CyclicBarrier```的代码修改一下就可以了：

```java
 //初始化一个计数为10的实例
 static final CyclicBarrier cyclicBarrier = new CyclicBarrier(10);
```

修改为：

```java
static Runnable command = new Runnable() {

    //清理函数
    void clean() {
        try {
            //模拟清理时的小号
            TimeUnit.SECONDS.sleep(2);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }

    @Override
    public void run() {
        System.out.println("所有线程运行完毕，线程：" + Thread.currentThread().getName() + "执行清理工作");
        clean();
    }
};

//初始化一个计数为10的实例
static final CyclicBarrier cyclicBarrier = new CyclicBarrier(10, command);
```

这样每次栅栏都突破后就会执行```command```。

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

其中```Generation```是内部类，代码很简单：

```java
private static class Generation {
    boolean broken = false;
}
```

我们还是从顶层的```await()```和```await(long timeout, TimeUnit unit)```方法的源码看起，代码如下：

```java
public int await() throws InterruptedException, BrokenBarrierException {
    try {
        return dowait(false, 0L);
    } catch (TimeoutException toe) {
        throw new Error(toe); // cannot happen
    }
}
```

```java
public int await(long timeout, TimeUnit unit)
    throws InterruptedException,
           BrokenBarrierException,
           TimeoutException {
    return dowait(true, unit.toNanos(timeout));
}
```

可以看到代码很简单，逻辑都在```dowait(boolean timed, long nanos)```这里，源码：

```java
private int dowait(boolean timed, long nanos)
    throws InterruptedException, BrokenBarrierException,
           TimeoutException {
    final ReentrantLock lock = this.lock;
    //需要先获得锁
    lock.lock();
    try {
        final Generation g = generation;
        //如果栅栏已经被突破就抛出异常
        //正常情况下运行到这里栅栏不应该被突破的
        if (g.broken)
            throw new BrokenBarrierException();
        
        //如果线程的中断标记已被设置，就突破屏障并抛出异常。
        //和Object#wait、ConditionObject#await、Thread#sleep类似，
        //如果调用前就设置了中断标志则会直接抛出异常
        //有个不同的是LockSupport#park，该方法并不响应中断
        if (Thread.interrupted()) {
            //有线程被中断的话就突破栅栏
            //栅栏被突破会唤醒所有在等待的线程，被唤醒的线程会
            //抛出BrokenBarrierException
            breakBarrier();
            throw new InterruptedException();
        }
        
        //如果以上异常均没有发生，就进入真正等待的逻辑了
        int index = --count;
        //本线程执行到await后，等待在await处的线程已经达到了parties的值
        //这个时候就要唤醒所有等待在await的线程了
        if (index == 0) {  // tripped
            //执行我们设置的command是否成功的标记，默认是失败的
            boolean ranAction = false;
            try {
                final Runnable command = barrierCommand;
                if (command != null)
                    command.run();
                //我们设置的command成功执行完毕，直接进入下一代即可
                ranAction = true;
                nextGeneration();
                return 0;
            } finally {
                //如果我们的command有问题，抛出了异常
                //为了避免产生死锁必须突破屏障
                if (!ranAction)
                    breakBarrier();
            }
        }
        
        //index不为0的话就进入下面的逻辑等待，也就是说下面的代码就是真正的等待逻辑
        //可以看到是一个自旋
        // loop until tripped, broken, interrupted, or timed out
        for (;;) {
            try {
                //判断是否设置超时时间，选择不同的等待方法
                if (!timed)
                    //trip是AQS的内部类ConditionObject的实例
                    //执行这句方法后，当前线程会释放锁并阻塞
                    trip.await();
                else if (nanos > 0L)
                    //和上面类似，不过带有超时时间
                    nanos = trip.awaitNanos(nanos);
            } catch (InterruptedException ie) {
                //等待过程中被中断，并且栅栏没有被突破
                //这种情况下先突破栅栏，然后抛出异常
                if (g == generation && ! g.broken) {
                    breakBarrier();
                    throw ie;
                } else {
                    //执行到这里，说明栅栏已经执行到下一代或者已经被突破了
                    //也就是g != generation 或者 g.broken 这两个条件有一个满足了
                    //如果上面的两个条件满足了，说明当前线程逻辑上已经完成了在栅栏上的等待了
                    //但是因为调度原因还没有从await醒来，这个时候被其他线程中断了
                    //逻辑上来说不应该抛出异常了，所以这里只是把中断补上
                    // We're about to finish waiting even if we had not
                    // been interrupted, so this interrupt is deemed to
                    // "belong" to subsequent execution.
                    Thread.currentThread().interrupt();
                }
            }

            //如果栅栏因为breakBarrier()方法执行了而被突破
            //那么其他等待的线程抛出该异常
            if (g.broken)
                throw new BrokenBarrierException();

            //如果栅栏已经进入到下一代了
            if (g != generation)
                return index;

            //可以看到等待超时的话也会突破栅栏并抛出TimeoutException
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

```breakBarrier```和```nextGeneration```方法的代码如下：

```java
private void breakBarrier() {
    generation.broken = true;
    //重置阻塞线程数
    count = parties;
    //唤醒所有在此Condition上等待的线程
    trip.signalAll();
}

private void nextGeneration() {
    // signal completion of last generation
    trip.signalAll();
    // set up next generation
    count = parties;
    generation = new Generation();
}
```

从上面的源码可以看出```dowait```方法的逻辑了：

1. 首先该方法是响应中断的，在栅栏处等待时被中断会打破栅栏并抛出```InterruptedException```

2. 在栅栏处等待的线程，除了因为超时抛出```TimeoutException```和上面的```InterruptedException```外，其余因为某个阻塞在栅栏处的线程的异常而唤醒的线程会抛出```BrokenBarrierException```

3. 正常情况下，阻塞在栅栏处的线程会在所有线程到达后执行我们设定的```command```（如果设定了的话），并且如果我们的```command```抛出了异常的话，其余线程也是会抛出```BrokenBarrierException```的，执行```command```的线程会抛出```command```中的异常。没有发生任何异常的话，栅栏进入下一代，继续拦截线程。

上面就是```dowait```方法的逻辑，可以看到是用等待通知模式完成的，源码并不算复杂。

最后看一下```reset```方法：

```java
public void reset() {
    final ReentrantLock lock = this.lock;
    lock.lock();
    try {
        breakBarrier();   // break the current generation
        nextGeneration(); // start a new generation
    } finally {
        lock.unlock();
    }
}
```

可以看到```reset```方法其实就是先突破屏障，然后进入下一代。如果栅栏因为异常被突破后还想继续使用栅栏的话，应该使用```reset```方法来把栅栏重置，否则所有新到达栅栏处的线程都会抛出```BrokenBarrierException```异常，并且如果有旧的线程在栅栏处等待的时候执行了```reset```方法的话，也会抛出```BrokenBarrierException```异常。