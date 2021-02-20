[TOC]
# 1.AQS简介
> *本文源码基于JDK8*。因为本人水平有限，错误和不足之处在所难免，欢迎指出错误和不足之处，一起进步。
>
> *这篇文章比较长，涉及到AQS的都放在这篇博客里了，暂时不打算看的部分可以直接跳过。*

  首先从大局上介绍一下AQS和一些相关的知识，这部分对后面阅读源码有帮助，熟悉这些概念的同学可以大致浏览一遍，有关MCS锁和CLH锁的部分可以直接跳过选择不看。
  **AbstractQuenedSynchronizer**，简称AQS。从名字就能看出AQS是一个**_抽象的_**、***基于队列***的同步器，这里的抽象的并不是说AQS是由abstract关键字修饰的，而是因为AQS是不能直接拿来用的，需要我们实现一些方法才能使用。AQS本身是作为框架使用的，juc（java.util.concurrent）包中很多同步工具比如ReentrantLock、CountDownLatch、Semphore、ReentrantReadWriteLock、FutureTask等类都是基于AQS来实现的，这几个工具都有作为同步工具的AQS的子类```Sync extends AbstractQueuedSynchronizer```。AQS提供了对内部资源state的原子性管理以及对线程调度的管理。
AQS内部主要有一个**变量state、一个严格FIFO的CLH队列、内部类Node**和**ConditionObject**。

1. **volatile的变量state**：该变量可以抽象的理解为资源的数量，比如在ReentrantLock中该变量就可以表示为是否获得锁以及获得锁的次数。AQS提供了三种方式来修改state的值，```protected final int getState()```、```protected final void setState(int newState)```、```protected final boolean compareAndSetState(int expect, int update) ```，子类可以直接调用这三个方法但不能重写这三个方法。

2. **内部类Node**：用于封装线程,添加了一些有用的状态标识信息，例如当前节点是否为独占模式、等待状态、前驱后去节点的引用。获取资源失败的线程会被封装为Node然后加入到CLH同步队列中。

3. **先进先出（FIFO）的CLH队列**：当线程尝试获取资源失败时就会加入到这个队列中，等待调度。  
    同步队列主要有两个选择，一个是MCS锁（Mellor-Crummey和Scott锁）的变体，另一种就是CLH锁（Craig，Landin和Hagersten锁）的变体。传统的MCS锁和CLH锁都是自旋锁，自旋锁的意思就是没法获得锁的线程并不会被阻塞，而是一直循环，直到锁可用。  
    
    ```java
    while(canNotGetTheLock) {
      //让出cpu
    }
    getLock();
    ```
    好处是避免了线程阻塞和唤醒的开销，在竞争不激烈，线程可以很快获得锁的情况下，这种方式可以很快的获得锁，从而提高了性能，但是，当锁竞争非常激烈时，这种方式可能会使得cpu一直空转执行while循环，浪费了cpu资源，反而降低了程序性能。
    MCS锁：基于单向链表的公平锁，申请锁的线程在本地变量上自旋，节点的前驱负责通知线程结束自旋。 
    ```java
    //申请锁的线程在本地变量blocked上自旋
    while(curNode.blocked) {
        //让出cpu
    }
    //前驱的节点通知后续节点结束自旋
    nextNode.blocked = false;
    ```
    CLH锁：基于隐式单链表的公平锁，申请锁的线程在前驱节点的本地变量上自旋。

    ```java
    //申请锁的线程在前驱的本地变量上自旋
    while(pred.status == locked) {  //表示前驱节点仍占用着锁,在传统的CLH队列中，这个pred并不是真正的引用，这里只是方便理解才加上了pred，
        //让出cpu                    //实际上应为status == locked，而这个status就是前驱的本地变量status，因为我们只需要在前驱的status上自旋，
    }                               //实际上并不需要前驱节点的引用
    //前驱节点使用完锁
    this.status = unlocked;
    ```
    对MCS锁和CLH锁感兴趣的同学可以深入去了解一下，如果只是看AQS源码的话，大致了解一下这些知识就已经足够了。 
    
    AQS中CLH队列实现是一个带有头节点的双向链表的数据结构，每个节点显式的保存前驱节点的引用```prev```和后驱节点的引用```next```，一个节点对应一个线程，只有获取资源（tryAcquire/tryAcquireShared）失败的线程，才会被封装为Node并且进入同步队列进行调度，直接获取成功的线程是不会进入同步队列进行调度的。需要注意的是head指向的节点是没有对应线程的，换句话说就是如果一个节点成为了头节点么这个节点对应的线程已经成功获得了资源了，且同步队列中只有头节点的下一个结点会尝试执行tryAcquire(Shared)来获得资源，而其他的节点是不会尝试获得资源的。
    
    AQS本身保存了头节点和尾节点的引用```private transient volatile Node head```和```private transient volatile Node tail```

  [![ssLKFs.png](https://s3.ax1x.com/2021/01/17/ssLKFs.png)](https://imgchr.com/i/ssLKFs)

4. **ConditionObject**：用于提供条件等待队列的支持，提供类似于Object类的wait和notify的api，内部维护了一个单链表，调用了await的线程会释放已经获取的资源并加入到该链表中。当有其他线程调用signal时，被唤醒的线程（可能是多个）会重新加入CLH同步队列中参与资源的争夺。也就是说AQS中是有两种队列的：一种用于同步调度的CLH**同步队列**，令一种是用于实现Condition的**等待队列**。ConditionObject与锁配合使用，可以实现类似java管程（Object的wait/notify）的功能，即下面代码表示的功能。

   ```java
   //线程1
   synchronized(locker) {
       ...
       while(!condition) //条件不满足
           locker.wait();
       ...
   }
   //线程2
   synchronized(locker) {
       ...
       locker.notify();
       ...
   }
   ```

   调用```Object#wait```和```Object#notify```前需要先获得对应对象的监视器锁，也就是```synchronized(locker)```这段代码，否则会抛出```IllegalMonitorStateException```异常。```ConditionObject```的```await()```和```signal()```方法在使用也是需要先获得锁，否则会（也应该）抛出```IllegalMonitorStateException```异常。

    > 需要注意的是，```await()```方法抛出该异常是依赖于```tryRelease(int arg)```方法的，因此我们在使用AQS构建独占模式的同步工具并且需要Condition时，当重写```tryRelease```方法的时候，应该要先判断当前线程是否已经获得锁，没有获得锁的话应该抛出异常。其实理论上来说独占模式下```tryRelease```都应该（可以）加上这样的判断，当不使用Condition且完全相信自己代码健壮性时也可以不加这个判断。

## 1.1内部类Node

内部类Node比较简单，源码如下：

 ```java
static final class Node {
        //用于标识当前节点为共享模式
        static final Node SHARED = new Node();
        //用于标识当前节点为独占模式
        static final Node EXCLUSIVE = null;

        //waitStatus的具体值，用于指示当前节点已被取消，被取消的节点不会再进入其他状态，AQS中有些地方通过判断waitStatus是否大于0来判断节点状态
        static final int CANCELLED =  1;
        //waitStatus的具体值，表示当前节点释放资源后需要唤醒后继有效结点
        static final int SIGNAL    = -1;
        //waitStatus的具体值，指明当前节点在某个CONDITION上等待
        static final int CONDITION = -2;
        /**
         * waitStatus的具体值，表明下一个acquireShared应该无条件的传播，
         * 因为在共享模式下，可能有多个线程会同时获得资源，也有可能某个线程释放
         * 的资源个数可以供多个同步队列中的节点获取，因此可能需要唤醒多个同步
         * 队列中的线程
         */
        static final int PROPAGATE = -3;
        //线程的等待状态
        volatile int waitStatus;  
        //前驱节点的引用
        volatile Node prev; 
        //后驱节点的引用
        volatile Node next; 
        //被封装的线程的引用
        volatile Thread thread;   
        /**
         * 这个引用有两个用处，当节点位于同步队列中时，nextWaiter用于标识线程是共享
         * 模式还是独占模式，当节点位于等待队列中时，nexWaiter用于保存下一个等待
         * 节点的引用
         */
        Node nextWaiter; 
        
        //判断节点是否为共享模式
        final boolean isShared() {
            return nextWaiter == SHARED;
        }
    
        //获得直接前驱
        final Node predecessor() throws NullPointerException {
            Node p = prev;
            if (p == null)
                throw new NullPointerException();
            else
                return p;
        }
    
        Node() {    // Used to establish initial head or SHARED marker
        }
    
        Node(Thread thread, Node mode) {     // Used by addWaiter
            this.nextWaiter = mode;
            this.thread = thread;
        }
    
        Node(Thread thread, int waitStatus) { // Used by Condition
            this.waitStatus = waitStatus;
            this.thread = thread;
        }
    }
 ```
关键的属性和相应的作用如下表：

| 属性 |描述|
|:---:|:----|
|waitStatus|表示了当前节点的状态，总共有5种状态。<br />  1. 0值，代表初始状态或者中间状态<br />  2. SIGNAL，代表当前节点需要在释放资源后唤醒一个后继节点来获取资源<br />  3. CONDITION，代表节点此时在等待队列中而不在同步队列中<br />  4. PROPAGATE，代表下一个acquireShared应该无条件传播，在```shouldParkAfterFailedAcquire```方法中体现<br />  5. CANCELLED， 代表当前节点已经被取消，此状态是唯一一个大于0的状态|
|prev|前驱节点的引用，prev是通过CAS的方式来**原子性**的插入链表的|
|next|后驱节点的引用，next的设置并非原子性的，**仅作为一种优化手段**，如果一个节点的next字段为null，并不一定表示该节点没有后继节点了，总是可以通过prev来向前访问，查看是否有有效后继节点（这里如果有疑问的话暂时不用深究，看到源码就明白了）|
|nextWaiter|当节点位于同步队列中时，nextWaiter用于标识线程是共享<br />当节点位于阻塞队列时，nextWaiter用于保存下一个节点的引用|

## 1.2内部类ConditionObject
  ```ConditionObject```实现了```java.util.concurrent.locks.Condition```接口,提供了类似Object类的wait和notify（java管程）类似的api。ConditionObject**只能在独占模式下使用**。与Object类的wait/notify相似，**Condition需要与锁一起使用**，在调用```Condition#await```和```Condition#signal```（以及对应的其它版本的api，比如带超时时间的await）之前需要先获得锁，否则会抛出```IllegalMonitorStateException```。一个锁可以与多个```ConditionObject```绑定，这是AQS比Java自带的wait/notify更加强大的地方，Java自带的api只能实现一个监视器锁对应一个condition。

  调用```ConditionObject#await```的线程会被封装为Node实例添加到等待队列中，当线程被唤醒或者被中断时会被移到同步队列上。因为是通过```LockSupport.park```来实现等待时阻塞的，因此线程被唤醒可能是因为调用了```LockSupport#unpark```或者```Thread#interrupt```方法。如果线程是在被```signal```方法唤醒之前中断的话，根据ConditionObject的逻辑会抛出```InterruptedException```，而如果线程是在被```signal```方法唤醒之后被中断的话，则不会抛出异常，只是重新设置线程的中断标志为true。在这里可以先不用管具体如何实现的，在清楚的了解了AQS独占模式下工作的流程后再看对应的实现会事半功倍，因为节点从等待队列转移到同步队列以及之后在同步队列中调度的过程和独占模式下AQS中调度过程是完全相同的。

# 2.AQS的使用
当我们使用AQS构建同步工具时，需要重写以下方法：
1. 独占模式下需要重写：
    - ```protected boolean tryAcquire(int arg)```,获取资源成功则返回true，否则返回false
    - ```protected boolean tryRelease(int arg)```，释放资源成功返回true，否则返回false
    - ```protected boolean isHeldExclusively()```这个方法只有需要用到Condition的时候才需要重写，如果当前线程已经获得锁返回true，否则返回false
2. 共享模式下需要重写：
    - ```protected int tryAcquireShared(int arg)```，返回值为本次获取成功之后仍剩余的资源数目
    
    - ```protected boolean tryReleaseShared(int arg) ```，释放资源成功返回true，否则返回false
    

  AQS源码中这几个方法默认是直接抛出异常的```throw new UnsupportedOperationException()```，没有把它们定义成abstract方法而是直接抛异常的原因可能是因为我们使用AQS时一般只会使用独占模式或者共享模式，而如果把这些方法都定义为abstract方法的话，我们在使用AQS构建同步工具的时候就需要把这几个方法都实现，所以不如直接抛异常。

## 2.1使用AQS构建互斥锁（mutex）
```java
import java.util.concurrent.TimeUnit;
import java.util.concurrent.locks.AbstractQueuedSynchronizer;
import java.util.concurrent.locks.Condition;
import java.util.concurrent.locks.Lock;

public class Mutex implements Lock {

    private final Sync sync;

    static class Sync extends AbstractQueuedSynchronizer {

        @Override
        protected boolean tryAcquire(int arg) {
            if (compareAndSetState(0, 1)) {
                setExclusiveOwnerThread(Thread.currentThread());
                return true;
            }
            return false;
        }

        @Override
        protected boolean tryRelease(int arg) {
            if (getExclusiveOwnerThread() != Thread.currentThread()) {
                throw new IllegalMonitorStateException();
            }
            setState(0);
            setExclusiveOwnerThread(null);
            return true;
        }

        @Override
        protected boolean isHeldExclusively() {
            return Thread.currentThread() == getExclusiveOwnerThread();
        }
    }


    public Mutex() {
        this.sync = new Sync();
    }

    @Override
    public void lock() {
        sync.acquire(1);
    }

    @Override
    public void lockInterruptibly() throws InterruptedException {
        sync.acquireInterruptibly(1);
    }

    @Override
    public boolean tryLock() {
        return sync.tryAcquire(1);
    }

    @Override
    public boolean tryLock(long time, TimeUnit unit) throws InterruptedException {
        return sync.tryAcquireNanos(1, unit.toNanos(time));
    }

    @Override
    public void unlock() {
        sync.release(1);
    }

    @Override
    public Condition newCondition() {
        return sync.new ConditionObject();
    }
}


```


# 3.AQS源码
这部分如果看着很乱摸不着头脑或者有的地方不明白的话，不必深究，多看几遍，对AQS整体有一定的掌握后就会明白的。因为AQS确实有一点复杂（或者说有很多不大好想的细节）。
## 3.1acquire
```acquire(int arg)```为独占模式下获取资源的顶级入口，arg为获取资源的数量。
```java
public final void acquire(int arg) {
        if (!tryAcquire(arg) &&
            acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
            selfInterrupt();
    }
```
如果```tryAcquire(arg)```获取资源成功的话，会直接返回，否则的话才会去执行```addWaiter(Node.EXCLUSIVE), arg)```以及之后的```acquireQueued```方法。  
```addWaiter(Node.EXCLUSIVE), arg)```这个方法的作用是把线程封装为内部类Node的**独占模式**的实例，并且把该实例添加到同步队列中，代码如下

```java
private Node addWaiter(Node mode) {
        Node node = new Node(Thread.currentThread(), mode);
        // Try the fast path of enq; backup to full enq on failure
        Node pred = tail;
        //tail不为null则尝试将node添加到同步队列
        if (pred != null) {
            node.prev = pred;
            //仍然是使用CAS的方式原子性的修改tail，保证只有一个线程能成功修改tail
            if (compareAndSetTail(pred, node)) {
                pred.next = node;
                return node;
            }
        }
        //如果上面快速入队失败则进入正常的入队流程
        enq(node);
        return node;
    }
```
addWaiter方法其实就是```enq(Node node)```的快速版本，“Try the fast path of enq; backup to full enq on failure”这句注释可能是因为直接调用``` enq(node)```的话可能多了’很多‘工作，所以在addWaiter方法中先尝试直接入队，失败了再进入```enq(node)```方法中入队。

```enq(Node node)```方法如下：

```java
private Node enq(final Node node) {
        for (;;) {
            Node t = tail;
            //当tail为null的时候，说明队列为空，此时需要初始化一个头节点
            if (t == null) { // Must initialize
                //因为可能有多个线程尝试初始化队列，因此需要用CAS的方式设置头节点，保证只有一个线程能初始化成功
                if (compareAndSetHead(new Node()))
                    tail = head;
            } else {
                node.prev = t;
                //同样因为可能有多个线程会尝试将自己的node实例加入同步队列尾部，因此需要使用CAS的方式设置队尾，保证只有一个线程设置成功
                if (compareAndSetTail(t, node)) {
                    t.next = node;
                    return t;
                }
            }
        }
    }
```
可以看出else代码块中的逻辑其实就是addWaiter方法中快速入队的逻辑。

  这个方法中需要注意的是，当同步队列为空时（t == null的话，同步队列一定为空），需要先构建一个头节点，构建的这个头节点是没有封装线程的（因为成为头节点的节点是否封装线程已经没有意义了），并且成功构建头节点的线程并不会从该方法中返回，而是进入下一个循环执行else对应的代码，尝试把自己的node添加到同步队列中。

至此，```addWaiter(Node.EXCLUSIVE), arg)```执行完毕，线程已经成功的被封装为Node实例并加入到了同步队列，那么接下来的工作就是如何参与调度了，也就是执行``` acquireQueued(addWaiter(Node.EXCLUSIVE), arg)```，这个方法就是获取资源失败的线程参与调度的主要方法，代码如下：

```java
final boolean acquireQueued(final Node node, int arg) {
        boolean failed = true;
        try {
            //中断标记，从这里也能看出acquire是不响应中断的
            boolean interrupted = false;
            for (;;) {
                final Node p = node.predecessor();
                //只有头节点的第一个后继节点会执行tryAcquire来尝试获得资源，符合CLH同步队列FIFO的特性，其余节点会直接跳过这个if
                if (p == head && tryAcquire(arg)) {
                    setHead(node);
                    p.next = null; // help GC
                    failed = false;
                    return interrupted;
                }
                //先判断是否可以park阻塞自己了，不能park的话会直接进入下一个循环
                if (shouldParkAfterFailedAcquire(p, node) &&
                    parkAndCheckInterrupt())
                    interrupted = true;
            }
        } finally {
            if (failed)
                cancelAcquire(node);
        }
    }
```
从代码中可以看出：

1. 独占模式下**只有**当节点的前驱为头节点时，线程才会执行tryAcquire来获取资源，获取成功后会把自己设置为头节点，失败的话就和其它节点一样，继续循环参与调度。setHead方法的源码很简单，就是直接设置```head = node;```（并且设置头结点的thread引用和prev引用为null，来帮助GC回收没用的节点），因为只有一个线程（tryAcquire成功的那个线程，在这个线程执行release释放资源之前，独占模式下其它线程是不可能执行到setHead这里的）会执行到该方法，所以该方法是不需要进行同步的，也不需要使用CAS的方式来设置头节点。
2. 所有不能获取资源（不是头节点的直接后继或者虽然是直接后继但是tryAcquire没能成功）的节点，都会执行```shouldParkAfterFailedAcquire```方法，判断获取资源失败后是否可以park阻塞自己，这个代码逻辑也很简单，就是只有直接有效前驱的waitStatus为SIGNAL时，线程才能park自己，否则线程不能park自己（这是因为只有当前节点的ws为SIGNAL时，执行release时才会唤醒后继节点，否则的话是不会去唤醒后继节点的，因此需要把前驱的ws设为SIGNAL）。
```java
private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
        int ws = pred.waitStatus;
        //如果直接前驱的waitStatus已经为SIGNAL，则返回true，表示线程可以直接park自己了
        if (ws == Node.SIGNAL)
            /*
             * This node has already set status asking a release
             * to signal it, so it can safely park.
             */
            return true;
        //前驱已经被取消，需要跳过所有被取消的前驱，找到一个有效的直接前驱，这种情况下显然线程不可以park自己
        if (ws > 0) {
            /*
             * Predecessor was cancelled. Skip over predecessors and
             * indicate retry.
             */
            do {
                node.prev = pred = pred.prev;
            } while (pred.waitStatus > 0);
            pred.next = node;
        } else {
            /*
             * waitStatus must be 0 or PROPAGATE.  Indicate that we
             * need a signal, but don't park yet.  Caller will need to
             * retry to make sure it cannot acquire before parking.
             */
            //执行到这里，前驱的waitStatus只能为0或者PROPAGETE了，这两种情况下，我们都应该把前驱的ws设置为SIGNAL
            //其中PROPAGATE是共享模式下的状态，表示唤醒需要无条件传播，这里不用深究
            //独占模式下执行到这里，直接前驱的ws只能为0
            compareAndSetWaitStatus(pred, ws, Node.SIGNAL);
        }
        return false;
    }
```
这里节点的直接前驱ws为0有两种可能：
1. 前驱节点是初始状态，ws默认为0
2. 前驱节点为SIGNAL或PROPAGATE，但是执行```unparkSuccessor(Node node)```方法时被设置为了0
第一种情况下，线程理论上来说可以park自己了，因为这时候下次循环很大概率是获取不到资源的（典型的例子就是这个节点不是头节点的直接后继或者同步队列本身已经很长），但是第2种情况下，当前节点的直接前驱肯定是头节点且资源已经释放了（已经执行tryRelease成功了），当前线程已经可以tryAcquire成功了，因此不必阻塞，直接进入下一次循环。因为我们没有办法区分是因为第一种原因导致ws为0还是第二种原因导致的，所以没办法只让第一种情况的线程阻塞，让第二种情况的线程不阻塞，因此选择不直接阻塞线程。这里应该是一种优化，因为第一种情况下的线程很快就会再次执行到这个方法，之后会阻塞自己，避免了重复阻塞唤醒线程。
到这里，如果线程可以park自己了，就执行```parkAndCheckInterrupt()```方法，代码如下：
```java
private final boolean parkAndCheckInterrupt() {
        LockSupport.park(this);
        return Thread.interrupted();
    }
```
代码很简单，就是阻塞自己，被唤醒后返回中断标记。
最后，在acquireQueued中成功获取资源后返回中断标志，如果在排队的过程中线程被中断过，```acquire```方法就就将中断补上```selfInterrupt();```，该方法就一条语句```Thread.currentThread().interrupt()```。**acquire方法是不响应中断的**。

到这里，正常```acquire```方法就结束了，另外如果在排队调度的过程中发生异常的话，是会执行```cancelAcquire(node)```取消节点调度的。正常情况下如果我们重写的```tryAcquire```方法不会出现异常的话，是不会发生取消节点的情况的。

总结一下acquire的流程大致如下：

[![ykeND0.png](https://s3.ax1x.com/2021/01/30/ykeND0.png)](https://imgchr.com/i/ykeND0)

## 3.2release和releaseShared
```	release(int arg)```为独占模式下释放资源的顶层入口，返回值true表示释放资源成功

```java
public final boolean release(int arg) {
        if (tryRelease(arg)) {
            Node h = head;
            //头节点不为null并且ws为SIGNAL时才会唤醒后继节点
            //因为shouldParkAfterFailedAcquire方法只有把头节点的ws设为SIGNAL时才会park自己
            //因此如果head的ws不为SIGNAL的话就无需唤醒后继节点了
            if (h != null && h.waitStatus != 0)
                unparkSuccessor(h);
            return true;
        }
        return false;
    }
```

释放资源的逻辑仍在在我们重写的```tryRelease(arg)```中，如果执行```tryRelease(arg)```释放资源成功，并且头节点的waitStatus不为0的话，就唤醒后继结点，这里头节点的waitStatus如果不为0，那么就只能是SIGNAL。然后执行```unparkSuccessor(h)```唤醒后继节点来获取资源。

```java
private void unparkSuccessor(Node node) {
        /*
         * If status is negative (i.e., possibly needing signal) try
         * to clear in anticipation of signalling.  It is OK if this
         * fails or if status is changed by waiting thread.
         */
        int ws = node.waitStatus;
        //这里把头结点的waitStatus设置为0是一种优化手段，允许失败
        //那么这里的CAS什么情况下会失败呢？只有第二个节点几乎同时执行了shouldParkAfterFailedAcquire方法，
        //这时这里的CAS操作就有可能失败，这个时候这里的CAS失败没有影响，因为下一个线程很快就会被设置为头节点
        if (ws < 0)
            compareAndSetWaitStatus(node, ws, 0);

        /*
         * Thread to unpark is held in successor, which is normally
         * just the next node.  But if cancelled or apparently null,
         * traverse backwards from tail to find the actual
         * non-cancelled successor.
         */
        //下面的方法就是跳过所有被取消了的节点，找到node的有效直接后继节点，然后唤醒
        Node s = node.next;
        //这里为null好像只有一种情况，就是当前节点的后继已经入队了，已经执行到addWaiter方法，并且执行node.prev = pred
        //和compareAndSetTail(pred, node)成功，这时候这个节点实际上已经入队了，但是pred.next = node这条
        //语句可能还没有执行，所以node.next为null，其他情况下我没有发现为null的情况。
        //因为没有一种能同时把prev和next同时原子性的设置的方法，所以next只作为一种优化，很多地方都要从队尾往前遍历
        if (s == null || s.waitStatus > 0) {
            s = null;
            //不论是何种情况，从队尾开始找node的有效直接后继节点
            for (Node t = tail; t != null && t != node; t = t.prev)
                if (t.waitStatus <= 0)
                    s = t;
        }
        //如果有直接后继就唤醒，正常情况下都是有的
        if (s != null)
            LockSupport.unpark(s.thread);
    }
```

被唤醒的线程一般情况会从```parkAndCheckInterrupt()```方法中醒来，然后继续执行``` acquireQueued```中的逻辑，尝试执行```tryAcquire(arg)```（因为释放资源的节点是头节点），正常情况下就会获取资源成功，到这里```release```的流程就结束了。

release的流程比较简单（unparkSuccessor的逻辑上面代码上的注释都很全，这里就没展开，展开反而会使得图更复杂，不够清晰）：

[![yVPJ1I.png](https://s3.ax1x.com/2021/01/31/yVPJ1I.png)](https://imgchr.com/i/yVPJ1I)

## 3.3acquireShared

```acquireShared```是**共享模式**下获取资源的顶层入口，代码如下：  
```java
public final void acquireShared(int arg) {
        //和acquire一样，只有当无法获取资源的情况下才需要AQS进行调度
        if (tryAcquireShared(arg) < 0)
            //入队和调度的逻辑都在该方法里
            doAcquireShared(arg);
    }
```
```tryAcquireShared```返回值大于0的话，当前线程一定是获取资源成功的，因此直接返回就行了，不需要AQS进行同步调度。```tryAcquireShared```返回0的话，说明刚好把资源消耗光了，下一次获取的线程就需要AQS调度了（不考虑中间有其他线程释放了资源的情况）。真正的加入同步队列以及调度的逻辑都在```doAcquireShared(arg)```中，代码如下：

```java
private void doAcquireShared(int arg) {
    //以共享模式创建节点，并将节点添加到同步队列中
    final Node node = addWaiter(Node.SHARED);
    //失败标记
    boolean failed = true;
    try {
        //中断标记
        boolean interrupted = false;
        for (;;) {
            final Node p = node.predecessor();
            //只有前驱节点为head的节点才会尝试获得资源，所以共享模式下节点仍然是先进先出
            if (p == head) {
                //尝试获得资源
                int r = tryAcquireShared(arg);
                //获得资源成功，可能仍有剩余资源，也可能没有
                if (r >= 0) {
                    //设置当前节点为头节点，并且进行传播。这个方法是和acquire区别最大的地方
                    setHeadAndPropagate(node, r);
                    p.next = null; // help GC
                    if (interrupted)
                        selfInterrupt();
                    failed = false;
                    return;
                }
            }
            //线程是否可以park自己的判断逻辑，和acquire中的是同一个方法，因此只有前驱的ws为SIGNAL时才会park自己
            if (shouldParkAfterFailedAcquire(p, node) &&
                parkAndCheckInterrupt())
                interrupted = true;
        }
    } finally {
        if (failed)
            cancelAcquire(node);
    }
}
```
可以看出```doAcquireShared```方法与```acquireQueued```十分相似。  
```doAcquireShared```也是不响应中断的。该方法将需要同步的线程以共享模式添加到同步队列，之后进行同步调度，同步调度的逻辑与```acquireQueued```基本一致，主要不同的地方是```tryAcquireShared(arg)```尝试获取资源成功后的逻辑，即```setHeadAndPropagate```，代码如下：

```java
private void setHeadAndPropagate(Node node, int propagate) {
        //头节点有可能会改变，因此记录原来的头节点
        Node h = head; // Record old head for check below
        //设置当前节点为新的头节点
        setHead(node);
        /*
         * Try to signal next queued node if:
         *   Propagation was indicated by caller,
         *     or was recorded (as h.waitStatus either before
         *     or after setHead) by a previous operation
         *     (note: this uses sign-check of waitStatus because
         *      PROPAGATE status may transition to SIGNAL.)
         * and
         *   The next node is waiting in shared mode,
         *     or we don't know, because it appears null
         *
         * The conservatism in both of these checks may cause
         * unnecessary wake-ups, but only when there are multiple
         * racing acquires/releases, so most need signals now or soon
         * anyway.
         */
        //以下几种情况满足一种就唤醒后继或者设置传播状态
        //1. propagate > 0，这时仍然还有剩余资源可获取，因此需要无条件传播
        //2. 原头结点的waitStatus < 0，可能为SIGNAL或者为PROPAGATE
        //3. 新的头节点的waitStatus < 0，可能为SIGNAL或者为PROPAGATE
        //这里有可能产生多余的唤醒，不用深究，往后看
        if (propagate > 0 || h == null || h.waitStatus < 0 ||
            (h = head) == null || h.waitStatus < 0) {
            Node s = node.next;
            //如果s == null成立，说明有新的节点入队，且还没有设置next指针
            if (s == null || s.isShared())
                doReleaseShared();
        }
```
```h == null```和```(h = head) == null```在这里是不会成立的，执行到这个方法的话，同步队列肯定不为空，那么```Node h = head```这句代码执行的时候head肯定不为null，而h又保存了head指向的节点实例的引用，因此对应的实例不可能在这个方法执行时被GC，所以h == null不会判定成功；```(h = head) == null```显然更不可能成立了，因此这里只是防止出现NPE的写法，不用过于纠结这个地方。

这里先看一下```doReleaseShared()```的源码再去讨论这几个情况什么时候会成立。

```doReleaseShared```源码如下：

```java
private void doReleaseShared() {
    /*
     * Ensure that a release propagates, even if there are other
     * in-progress acquires/releases.  This proceeds in the usual
     * way of trying to unparkSuccessor of head if it needs
     * signal. But if it does not, status is set to PROPAGATE to
     * ensure that upon release, propagation continues.
     * Additionally, we must loop in case a new node is added
     * while we are doing this. Also, unlike other uses of
     * unparkSuccessor, we need to know if CAS to reset status
     * fails, if so rechecking.
     */
    for (;;) {
        Node h = head;
        //队列为空或者只有头节点了不需要进行唤醒等操作
        if (h != null && h != tail) {
            int ws = h.waitStatus;
            //头节点的ws为SIGNAL，这个时候需要唤醒后继
            if (ws == Node.SIGNAL) {
                //这里需要使用CAS的方式去改变头节点的ws，只有成功的那个线程才能唤醒后继线程
                //失败线程则进入下一次循环
                if (!compareAndSetWaitStatus(h, Node.SIGNAL, 0))
                    continue;            // loop to recheck cases
                unparkSuccessor(h);
            }
            //这里如果头节点的ws等于0，只有可能是因为已经有其它线程唤醒了后继节点
            //所以这里只是设置头节点的ws为PROPAGATE，保证唤醒能够传播
            else if (ws == 0 &&
                     !compareAndSetWaitStatus(h, 0, Node.PROPAGATE))
                continue;                // loop on failed CAS
        }
        //在这里如果头节点改变了的话就继续循环
        if (h == head)                   // loop if head changed
            break;
    }
}
```
首先，能执行到这个方法的话h肯定不为null。h如果和tail相同也就是队列中只有一个节点了，这个时候唤醒后继显然是没意义的。排除这种情况后就可以进入唤醒后继的逻辑了。

当头节点的waitStatus为SIGNAL时，和```acquire```中一样，节点的waitStatus为SIGNAL表示节点需要唤醒后继节点，因此先原子性的清除节点的SIGNAL状态，成功清除的线程负责唤醒后继节点，失败的线程则继续循环。这里的CAS失败只有可能是多个线程在执行```doReleaseShared```。

首先看到修改节点ws的只有	```unparkSuccessor```、```doReleaseShared```、```shouldParkAfterFailedAcquire```这几个方法（不考虑取消节点```cancelAcquire```以及等待队列中的操作，等待队列中的操作在逻辑上是和这几个方法起到一样的作用的），在执行这里的CAS时，显然后继节点不可能在执行```shouldParkAfterFailedAcquire```中的CAS。因为运行到这里时，头节点的ws已经为SIGNAL了，说明后继节点的CAS操作已经执行过了，在这里的CAS成功之前是不会再执行了；```unparkSuccessor```方法中的CAS就更不可能了，因为执行```unparkSuccessor```的一定是CAS的把SIGNAL状态清除的线程。所以，这里CAS失败的原因只有可能是有多个线程在这里发生了竞争。

因为在共享状态下，```acquireShared```和```releaseShared```都会调用```doReleaseShared```方法，这里发生了竞争是因为```acquireShared```流程中成功获得资源的线程在执行```setHeadAndPropagate```时判断需要唤醒后继，所以执行了```doReleaseShared```方法；同时，前面获取资源成功的线程在使用完资源后，执行```releaseShared```方法，释放资源成功后，执行```doReleaseShared```方法，这个时候就发生了竞争。也就是以下两种情况：

1. 执行```setHeadAndPropagate```时，propagate参数大于0，也就是还有资源可以获取；h.waitStatus < 0（旧的头节点）；head.waitStatus < 0（新的头节点）。这里小于0，可能为SIGNAL，也可能为PROPAGATE。
2. 线程在执行```releaseShared```，调用了```doReleaseShared```方法。  这时候说明又有新的资源可以获取了。

CAS的清除了头节点的SIGNAL状态的线程去唤醒头节点的后继，这个时候头节点的ws就为0了，其它线程在执行的时候就会进入到传播的逻辑，就会CAS的把头节点的ws从0设置为PROPAGATE。头节点的ws被设置为PROPAGATE后，后面的线程执行```setHeadAndPropagate```时```h.waitStatus < 0```就会判定为true，从而再执行```doReleaseShared```方法。也就是说这里把头节点的ws设为PROPAGATE的目的是为了**让后续被唤醒的线程检测到可能又有资源可以获取了，请尝试执行```doReleaseShared```方法来唤醒后继线程**。当旧的头节点h或者新的头节点head的ws为SIGNAL时，逻辑上我认为不需要唤醒后继节点，正如源码里的注释写的，这里可能会产生多余的唤醒。但是产生过多的唤醒并不会产生错误，并且被唤醒的线程是比较靠近头部的节点，很有可能马上就能获得资源了，根据LockSupport的逻辑，提前unpark线程也可以避免线程阻塞。

最后关于```if (h == head)  ```这里的判断，为什么头节点改变了就继续循环呢？因为头节点改变了的话，有可能我们想要传达的信息并没有传达给后继节点（仍然还有资源可以获取，可能是获取完剩余的，也有可能是新释放的，总之是把h的ws设置为了PROPAGATE），因此我们需要继续循环尝试设置新的头节点的状态，将我们的信息传达出去。

到这里共享模式的源码解读就基本完成了，释放资源的方法很简答，代码如下：

```java
public final boolean releaseShared(int arg) {
    if (tryReleaseShared(arg)) {
        doReleaseShared();
        return true;
    }
    return false;
}
```
可以看出,主要逻辑就是```doReleaseShared```,而上面已经解读该方法了,这里不再赘述。
最后，```acquireShared```和```releaseShared```的流程如下：

[![yI1BWV.png](https://s3.ax1x.com/2021/02/20/yI1BWV.png)](https://imgchr.com/i/yI1BWV)  
[![yI1sQU.png](https://s3.ax1x.com/2021/02/20/yI1sQU.png)](https://imgchr.com/i/yI1sQU)