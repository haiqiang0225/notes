[toc]
# ReentrantLock 可重入锁分析

> *本文源码基于JDK8*。因为本人水平有限，错误和不足之处在所难免，欢迎指出错误和不足之处，一起进步。

**ReentrantLock**是**juc**中提供的可重入独占锁，Java内建的```synchronized```关键字也是一个可重入的独占锁（监视器锁）。**ReentrantLock**和```synchronized```在功能方面非常相似，并且从JDK6之后两者的性能上没有很大的区别。**ReentrantLock**比```synchronized```关键字功能更丰富、更灵活。两者在使用上是十分相似的。两者的相同处和不同如下：

相同点：

1. 两者都是可重入的独占锁，这里所谓的可重入就是已经获得锁的线程在申请同一个锁时可以直接获取锁而不会产生死锁，独占指在同一时刻只允许单个线程持有锁。
2. 两者都能实现wait/notify机制，并且两者在该机制上的语义是相同的。在调用前都需要获得**对应的**锁，否则会抛出```IllegalMonitorStateException```异常。

不同点：

1. ```	synchronized```只能实现非公平锁，而**ReentrantLock**既可以实现公平锁也可以实现非公平锁。公平锁就是先申请锁的线程会比后申请锁的线程先获得锁，而非公平锁则没有这个要求。**ReentrantLock**默认是非公平锁，可以通过有参构造器选择公平锁。
2. ```synchronized```因为竞争导致获取不到锁而_处于等待_状态时，是不会响应中断的，而**ReentrantLock**可以选择响应中断，也可以选择不响应中断。
3. ```synchronized```不需要显式的获取、释放锁，而**ReentrantLock**需要显式的获取、释放。
4. ```synchronized```只能有一个condition，而**ReentrantLock**可以有多个。```synchronized```使用condition的方法是```obj.wait()/obj.notify()```（synchronized获得的锁为obj上的对象锁），而**ReentrantLock**实现condition需要使用AQS的内部类```ConditionObject```，该类提供了比```synchronized```的等待/通知机制更加丰富的api。

## synchronized关键字

```synchronized```是Java内建的锁，锁与对象是对应的，一个对象对应一个锁，称为监视器锁（Monitor Lock），```synchronized```关键字在字节码层面就是```monitorenter```和```monitorexit```两条字节码。```monitorenter```对应```lock()```方法，会尝试获取对象上对应的锁。```monitorexit```对应```unlick()```方法，会释放对象上对应的锁。监视器锁是通过类实例上称为**对象头**的数据结构来实现的，底层进行了包括偏向锁、轻量级锁、重量级锁的优化。偏向锁即为不加锁，可以避免执行CAS操作而陷入内核态带来的开销，这种情况只有在单个线程多次申请同一个监视器锁时才会出现，如果有多个线程竞争锁就会由偏向锁升级为轻量级锁。所谓轻量级锁就是通过CAS操作来原子性的修改对象头上的数据，成功修改的线程视为成功获得锁，失败的线程会自旋，这种情况下线程竞争还不是很激烈。当自旋到一定的次数仍然没有获得锁时，说明竞争很激烈，这种情况下可能很长时间内线程都不能成功获得锁，此时再自旋只会白白浪费了CPU资源，因此锁膨胀为重量级锁，重量级锁下竞争锁失败的线程会直接被阻塞等待唤醒。

### synchronized关键字的使用

```synchronized```关键字有三种用法：

1. 修饰类（静态）方法，锁对应的对象为当前类的Class实例

2. 修饰实例方法

3. ```synchronize```代码块

其中当```synchronized```修饰类方法时，锁对应的对象为当前类对应的Class类实例，修饰实例方法时，锁对应的对线为当前类实例，```synchronized```代码块的锁为括号中的对象。下面用代码来验证。代码很简单，```SynchronizedTest```包含四个方法：```staticF1()、staticF2()、nonStaticF1()、nonStaticF2()```，均使用```synchronized```关键字来修饰，不同之处只有输出语句。

```java
import java.util.concurrent.TimeUnit;

public class SynchronizedTest {

    public synchronized static void staticF1() {
        System.out.println("static f1 start");
        try {
            TimeUnit.SECONDS.sleep(2);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        System.out.println("static f1 finished");
    }

    public synchronized static void staticF2() {
        System.out.println("static f2 start");
        try {
            TimeUnit.SECONDS.sleep(2);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        System.out.println("static f2 finished");
    }

    public synchronized void nonStaticF1() {
        System.out.println("f1 start");
        try {
            TimeUnit.SECONDS.sleep(2);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        System.out.println("f1 finished");
    }

    public synchronized void nonStaticF2() {
        System.out.println("f2 start");
        try {
            TimeUnit.SECONDS.sleep(2);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        System.out.println("f2 finished");
    }
}

```

下面分不同情况运行：

1. 1

   ```java
   public static void main(String[] args) {
       SynchronizedTest obj = new SynchronizedTest();
       new Thread(SynchronizedTest::staticF1).start();
       new Thread(SynchronizedTest::staticF2).start();
   }
   ```

   输出如下：

   ```java
   static f1 start
   static f1 finished
   static f2 start
   static f2 finished
   ```

   ```staticF1()```与```staticF2()```方法永远都是交替运行，即一个方法运行完之后另一个方法才会执行，这是因为执行这两个方法的线程都在竞争```SynchronizedTest.class```这个Class类实例上的锁。那么怎么验证修饰静态方法是对应这个对象上的锁呢？很简单，我们可以利用Java内建的等待/通知机制。修改```SynchronizedTest```中的方法，如下：

   ```java
   public synchronized static void staticF1() {
       System.out.println("static f1 start");
       try {
           SynchronizedTest.class.wait();
           TimeUnit.SECONDS.sleep(2);
       } catch (InterruptedException e) {
           e.printStackTrace();
       }
       System.out.println("static f1 finished");
   }
   
   public synchronized static void staticF2() {
       System.out.println("static f2 start");
       try {
           TimeUnit.SECONDS.sleep(2);
           SynchronizedTest.class.notify();
       } catch (InterruptedException e) {
           e.printStackTrace();
       }
       System.out.println("static f2 finished");
   }
   ```

   再次运行刚才的main方法，结果如下：

   ```java
   static f1 start
   static f2 start
   static f2 finished
   static f1 finished
   ```

   并没有抛出```IllegalMonitorStateException```。而如果把```SynchronizedTest.class.wait()```改为```String.class.wait()```或者```其它实例.wait()```就会抛出该异常，因为调用```wait()/notify()```之前需要先获得对应的锁，这里并没有获得String类的Class实例对应的锁，因此会抛出异常。

2. 

## ReentrantLock分析