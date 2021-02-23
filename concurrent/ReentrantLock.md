[toc]

# ReentrantLock 可重入锁分析

> *本文源码基于JDK8*。因为本人水平有限，错误和不足之处在所难免，欢迎指出错误和不足之处，一起进步。
>
> 阅读这篇文章需要对AQS有一定的了解，虽然本篇文章大致介绍了AQS但还是强烈建议先去大致了解下AQS再来阅读本篇文章。

**ReentrantLock**是**juc**中提供的可重入独占锁，Java内建的```synchronized```关键字也是一个可重入的独占锁（监视器锁）。**ReentrantLock**和```synchronized```在功能方面非常相似，并且从JDK6之后两者的性能上没有很大的区别。**ReentrantLock**比```synchronized```关键字功能更丰富、更灵活。两者在使用上是十分相似的。两者的相同处和不同如下：

相同点：

1. 两者都是可重入的独占锁，这里所谓的可重入就是已经获得锁的线程在申请同一个锁时可以直接获取锁而不会产生死锁，独占指在同一时刻只允许单个线程持有锁。
2. 两者都能实现wait/notify机制，并且两者在该机制上的语义是相同的。在调用前都需要获得**对应的**锁，否则会抛出```IllegalMonitorStateException```异常。

不同点：

1. ```	synchronized```只能实现非公平锁，而**ReentrantLock**既可以实现公平锁也可以实现非公平锁。公平锁就是先申请锁的线程会比后申请锁的线程先获得锁，而非公平锁则没有这个要求。**ReentrantLock**默认是非公平锁，可以通过有参构造器选择公平锁。
2. ```synchronized```因为竞争导致获取不到锁而_处于等待_状态时，是不会响应中断的，而**ReentrantLock**可以选择响应中断，也可以选择不响应中断。
3. ```synchronized```不需要显式的获取、释放锁，而**ReentrantLock**需要显式的获取、释放。
4. ```synchronized```只能有一个condition，而**ReentrantLock**可以有多个。```synchronized```使用condition的方法是```obj.wait()/obj.notify()```（synchronized获得的锁为obj上的对象锁），而**ReentrantLock**实现condition需要使用AQS的内部类```ConditionObject```，该类提供了比```synchronized```的等待/通知机制更加丰富的api。

## 1.synchronized关键字简析

```synchronized```是Java内建的锁，锁与对象是对应的，一个对象对应一个锁，称为监视器锁（Monitor Lock），```synchronized```关键字在字节码层面就是```monitorenter```和```monitorexit```两条字节码。```monitorenter```对应```lock()```方法，会尝试获取对象上对应的锁。```monitorexit```对应```unlick()```方法，会释放对象上对应的锁。监视器锁是通过类实例上称为**对象头**的数据结构来实现的，底层进行了包括偏向锁、轻量级锁、重量级锁的优化。偏向锁即为不加锁，可以避免执行CAS操作而陷入内核态带来的开销，这种情况只有在单个线程多次申请同一个监视器锁时才会出现，如果有多个线程竞争锁就会由偏向锁升级为轻量级锁。所谓轻量级锁就是通过CAS操作来原子性的修改对象头上的数据，成功修改的线程视为成功获得锁，失败的线程会自旋，这种情况下线程竞争还不是很激烈。当自旋到一定的次数仍然没有获得锁时，说明竞争很激烈，这种情况下可能很长时间内线程都不能成功获得锁，此时再自旋只会白白浪费了CPU资源，因此锁膨胀为重量级锁，重量级锁下竞争锁失败的线程会直接被阻塞等待唤醒。

### 1.1synchronized关键字的使用

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

1. 单独运行两个静态方法或者单独两个实例方法

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

   并没有抛出```IllegalMonitorStateException```。而如果把```SynchronizedTest.class.wait()```改为```String.class.wait()```或者```其它实例.wait()```就会抛出该异常，因为调用```wait()/notify()```之前需要先获得对应的锁，这里并没有获得String类的Class实例对应的锁，因此会抛出异常。（上面的代码理论上有可能会产生死锁，这里只是为了验证获得的锁是对应哪个对象的）

   运行两个实例方法和以上的结果是一样的，这里不放出来了。

2. 同时运行类方法和实例方法

   ```java
   public static void main(String[] args) {
       SynchronizedTest obj = new SynchronizedTest();
       new Thread(SynchronizedTest::staticF1).start();
       new Thread(obj::nonStaticF1).start();
       new Thread(SynchronizedTest::staticF2).start();
       new Thread(obj::nonStaticF2).start();
   
   }
   ```

   输出如下：

   ```java
   static f1 start
   f1 start
   f1 finished
   static f1 finished
   f2 start
   static f2 start
   static f2 finished
   f2 finished
   ```

   可以看到，静态方法和实例方法分别是按顺序输出的，但静态方法和非静态方法之间是交替输出的，所以竞争的锁并不是同一个锁。

3. ```synchronized```修饰的代码块

   用法如下：

   ```java
   public void syncBlock() {
       synchronized (this) {
           System.out.println("syncBlock start");
           try {
               TimeUnit.SECONDS.sleep(2);
           } catch (InterruptedException e) {
               e.printStackTrace();
           }
           System.out.println("syncBlock finished");
       }
   }
   ```

   锁对应的对象即为this，也就是括号里的对象。

因此，使用```synchronized```需要注意是竞争的哪个锁。

## 2.ReentrantLock分析

**ReentrantLock**实现了```Lock```接口，提供了```lock()、lockInterruptibly()、tryLock()、tryLock(long time, TimeUnit unit)、unlock()、newCondition()```等api。

> 阅读ReentrantLock（或者说juc中其他同步工具）时，最好对AQS有一定的了解，AQS是个很重要的框架，了解AQS后阅读其他的同步工具会事半功倍。[AQS分析](https://blog.csdn.net/qq_19007335/article/details/113894632)

### 2.1ReentrantLock使用 

下面的代码是```ReentrantLock```作为锁最常用的方式

```java
final ReentrantLock lock = new ReentrantLock();
lock.lock();
try {
    //do something
} catch (Exception e) {
    e.printStackTrace();
} finally {
    lock.unlock();
}
```

为了防止我们的逻辑出现异常导致死锁，```lock()```方法应该放在```try```块之前调用，而```unlock()```方法应该放在```finally```块中，以保证能正常释放锁。

**ReentrantLock**的实现采用了组合模式，其中有三个内部类：```Sync、NonfairSync、FairSync```，加锁解锁功能都是依靠这三个内部类实现的。其中```NonfairSync```实现了非公平的锁获取操作，而```FairSync```则提供了公平的锁获取操作。公平锁和非公平锁的不同之处只有在锁的获取阶段，所以```Sync```类中定义了锁的释放操作，而```NonfairSync```和```FairSync```则分别提供了非公平和公平的锁获取操作。公平锁和非公平锁的区别在文章开头有提到。下面看一下源码是怎么实现的。

### 2.2```Sync```源码分析

```java
abstract static class Sync extends AbstractQueuedSynchronizer {
    private static final long serialVersionUID = -5179523762034025860L;
    //公共的接口，方便ReentrantLock中的方法在公平和非公平模式下都保持相同的调用
    abstract void lock();
    
    //这个实现放在Sync中而不是NonfairSync中的目的是为了给tryLock()方法调用
    final boolean nonfairTryAcquire(int acquires) {
        final Thread current = Thread.currentThread();
        //这里state是AQS中的变量，在这里表示获得锁的次数
        //state为0表示没有线程获得锁，为1表示线程第一次获得锁
        //为2表示已经获得锁的线程第二次获得锁，也就是重入了
        int c = getState();
        if (c == 0) {
            //只要state为0就直接CAS的尝试获得锁，不管AQS同步队列中是否有节点
            //因此这里是非公平的，即新来的线程可能会比早来的线程先获得锁
            //因为是CAS所以保证了只有一个线程能获得锁
            if (compareAndSetState(0, acquires)) {
                //获得锁的线程设置标记
                setExclusiveOwnerThread(current);
                //获取成功
                return true;
            }
        }
        //如果剩余资源不为0的话，要看申请锁的线程是不是已经获得锁了
        //因为只有获得锁的线程才可以重入，继续增加state的值
        else if (current == getExclusiveOwnerThread()) {
            int nextc = c + acquires;
            if (nextc < 0) // overflow
                throw new Error("Maximum lock count exceeded");
            //因为是独占的锁，这里不需要CAS，只需要简单的设置state的值即可
            setState(nextc);
            return true;
        }
        //不满足上面的情况都判断为失败，失败的线程会进入FIFO队列进行调度
        return false;
    }
    
    //释放锁的逻辑
    protected final boolean tryRelease(int releases) {
        //c是释放之后state的值
        int c = getState() - releases;
        //判断一下释放锁的线程是否已经获得锁了，这里不添加的话，对正常
        //获得锁释放锁的逻辑是没有影响的（因为是独占的），但是对Condition
        //有影响，即不获得锁调用await()方法的话，不会抛出IllegalMonitorStateException异常
        if (Thread.currentThread() != getExclusiveOwnerThread())
            throw new IllegalMonitorStateException();
        //锁是否完全释放的标志，即释放后state是否为0
        boolean free = false;
        //为0时其实就是把exclusiveOwnerThread设置为null
        if (c == 0) {
            free = true;
            setExclusiveOwnerThread(null);
        }
        setState(c);
        //这里注意只有完全释放锁时才会返回true
        //因为如果返回true的话，AQS会唤醒后继节点来获得锁，因此如果不是完全释放就不应该返回true
        return free;
    }

    //判断当前线程是否是独占线程
    protected final boolean isHeldExclusively() {
        // While we must in general read state before owner,
        // we don't need to do so to check if current thread is owner
        return getExclusiveOwnerThread() == Thread.currentThread();
    }

    //返回一个新的ConditionObject对象，ConditionObject是AQS的内部类
    //实现了Condition接口，提供了所有Condition相关的操作
    final ConditionObject newCondition() {
        return new ConditionObject();
    }

    // Methods relayed from outer class

    //获得锁的拥有线程
    final Thread getOwner() {
        return getState() == 0 ? null : getExclusiveOwnerThread();
    }
    //获得重入的次数
    final int getHoldCount() {
        return isHeldExclusively() ? getState() : 0;
    }
    //获得锁的状态
    final boolean isLocked() {
        return getState() != 0;
    }

    /**
     * Reconstitutes the instance from a stream (that is, deserializes it).
     */
    //序列化用的方法
    private void readObject(java.io.ObjectInputStream s)
        throws java.io.IOException, ClassNotFoundException {
        s.defaultReadObject();
        setState(0); // reset to unlocked state
    }
}
```

```Sync```继承了```AbstractQueuedSynchronizer类```，这里简单的介绍一下AQS。AQS本身维护一个FIFO的同步队列，并且实现了获取资源（资源在这里就是锁）失败后的调度逻辑。使用AQS构建独占模式的同步器时，需要重写```tryAcquire、tryRelease、isHeldExclusively```方法，这几个方法在AQS本身中的实现是简单的抛出异常，也就是说AQS只是完成了线程的调度，但是获取和释放资源的逻辑仍然需要由我们来定义。并且AQS的内部类```ConditionObject```实现了等待/通知机制所需要的全部逻辑，因此在```ReentrantLock```中并没有```Condition```相关的实现逻辑。因为```ConditionObject```的实现也是与AQS密切相关的，因此在这里不做深入探究。

不难看出，```Sync```中主要是实现了```nonfairTryAcquire```和```tryRelease```的逻辑。在```Sync```中实现```nonfairTryAcquire```而不是在```NonfairSync```中实现的原因是因为在```tryLock```方法中调用了该方法，如果在```NonfairSync```实现的话，在公平模式下也需要创建一个```NonfairSync```实例，所以在```Sync```中实现该方法并没有问题。

看完了```Sync```的源码我们再来分别看一下```NonfairSync```和```FairSync```的源码。

### 2.3```NonfairSync```源码分析

```java
static final class NonfairSync extends Sync {
    private static final long serialVersionUID = 7316153563782823691L;

    /**
     * Performs lock.  Try immediate barge, backing up to normal
     * acquire on failure.
     */
    //在这里会先CAS的尝试获得锁，失败后进入正常的acquire逻辑
    final void lock() {
        if (compareAndSetState(0, 1))
            setExclusiveOwnerThread(Thread.currentThread());
        else
            acquire(1);
    }
    
    //只是调用nonfairTryAcquire
    protected final boolean tryAcquire(int acquires) {
        return nonfairTryAcquire(acquires);
    }
}
```

```NonfairSync```的代码非常简单，其实只有```lock()```的实现。

在这里同样不管是否AQS的同步队列中是否有等待调度的线程，都会CAS的尝试获得锁，这也是为什么这里是不公平的获取操作。如果这里获取失败了就进入正常的```acquire```逻辑，在```acquire```中会先调用```tryAcquire```方法，而这里的```tryAcquire```其实就是```nonfairTryAcquire```，所以还有可能进行一次CAS操作（state为0的情况下），如果成功就会比队列中等待的线程先获得锁，如果失败就会进入AQS的调度逻辑中。关于AQS是如何调度的这里不是重点因此不做深入。

### 2.4```FairSync```源码分析

```java
static final class FairSync extends Sync {
    private static final long serialVersionUID = -3000897897090466540L;

    //因为AQS本身的实现是公平的调度，因此直接进入AQS的acquire逻辑即可
    final void lock() {
        acquire(1);
    }

    /**
     * Fair version of tryAcquire.  Don't grant access unless
     * recursive call or no waiters or is first.
     */
    //公平模式下获得锁的逻辑，和非公平的相似
    protected final boolean tryAcquire(int acquires) {
        final Thread current = Thread.currentThread();
        int c = getState();
        //仍然是state为0时才会尝试获得锁
        if (c == 0) {
            //跟非公平模式不同之处在于这里需要判断AQS中是否仍有节点在排队
            //如果还有节点排队那么就不能CAS的去获取锁，需要进入AQS的同步队列中排队
            if (!hasQueuedPredecessors() &&
                compareAndSetState(0, acquires)) {
                //获取成功的逻辑
                setExclusiveOwnerThread(current);
                return true;
            }
        }
        //如果申请锁的线程已经获得锁了，就重入
        else if (current == getExclusiveOwnerThread()) {
            //重入的逻辑与非公平模式实现是一样的
            int nextc = c + acquires;
            if (nextc < 0)
                throw new Error("Maximum lock count exceeded");
            setState(nextc);
            return true;
        }
        //如果锁被完全释放但是AQS的同步队列中有线程在排队等待获得锁，或者锁仍未被完全释放则返回false
        //结果都是进入AQS的同步队列中进行调度
        return false;
    }
}
```

因为AQS中的同步队列是FIFO的，所以AQS本身的实现是公平的。代码上的注释写的很清楚了。可以看出公平模式下，只有锁完全被释放且AQS的同步队列中没有线程在等待锁时才会CAS的尝试获得锁，如果这个时候出现竞争，那么失败的线程会进入同步队列调度。重入的逻辑就不用解释了。其实这基本等同于直接在AQS中调度了，因为AQS中的同步队列是FIFO的，所以这里也就保证了获得锁的顺序基本上与申请锁的顺序相同的（如果CAS操作时发生竞争，那么我们认为CAS操作成功的那个线程是先申请锁的话）。

到这里，我们知道了公平模式和非公平模式如何实现的了，剩下的工作就很简答了。

### 2.5其他api以及实现

**ReentrantLock**本身也是锁，所以最重要的应该是加锁解锁的代码。

```java
//不响应中断，最终会成功获得锁
public void lock() {
    sync.lock();
}
//响应中断，最终会成功获得锁
public void lockInterruptibly() throws InterruptedException {
    sync.acquireInterruptibly(1);
}
//不响应中断，尝试获得锁，可能失败，失败后立即返回false
public boolean tryLock() {
    return sync.nonfairTryAcquire(1);
}
//响应中断，带有超时时间的获得锁，超时后返回false
public boolean tryLock(long timeout, TimeUnit unit)
        throws InterruptedException {
    return sync.tryAcquireNanos(1, unit.toNanos(timeout));
}
//释放锁的逻辑
public void unlock() {
    sync.release(1);
}
//返回AQS内部类ConditionObject的实例
public Condition newCondition() {
    return sync.newCondition();
}
```

可以看出，```ReentrantLock```加锁解锁的实现都非常简单，只是简单的调用了sync对应的方法。这是因为AQS本身把这些方法都实现了，包括响应中断的acquire、不响应中断的acquire、带有超时时间的acquire，而我们只需要定义```tryAcquire```和```tryRelease```的逻辑，因此显得非常简单。这就像我们在使用Spring框架时只需要写Controller、Service等代码一样，复杂的逻辑都由框架代替我们做了，我们才会觉得简单。这里还是强烈建议把AQS彻底搞明白，可以说AQS是构建并发包的基石，彻底明白了AQS是如何工作的以后，阅读其他同步工具的源码将会非常简单！