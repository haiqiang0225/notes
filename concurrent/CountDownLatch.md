[toc]

# CountDownLatch源码分析

> *本文源码基于JDK8*。因为本人水平有限，错误和不足之处在所难免，欢迎指出错误和不足之处，一起进步。
> 阅读这篇文章最好对AQS有一定的了解。

## 1.CountDownLatch介绍

**CountDownLatch**是juc中提供的同步工具。```CountDownLatch```就像是一扇门，

## 2.CountDownLatch使用

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
            countDownLatch.await();
            System.out.println("所有线程加载完毕，主线程继续运行");
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
}
```

## 3.CountDownLatch源码



