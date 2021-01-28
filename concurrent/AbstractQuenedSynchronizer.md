[TOC]
# 1.AQS简介
> *本文源码基于JDK8*。因为本人水平有限，错误和不足之处在所难免，欢迎指出错误和不足之处，一起进步。

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
|prev|前驱节点的引用，prev是通过CAS的方式来原子性的插入链表的|
|next|后驱节点的引用，next的设置并非原子性的，仅作为一种优化手段，如果一个节点的next字段为null，并不一定表示该节点没有后继节点了，总是可以通过prev来向前访问，查看是否有有效后继节点|
|nextWaiter|当节点位于同步队列中时，nextWaiter用于标识线程是共享<br />当节点位于阻塞队列时，nextWaiter用于保存下一个节点的引用|

## 1.2内部类ConditionObject
```ConditionObject```实现了```java.util.concurrent.locks.Condition```接口,提供了类似Object类的wait和notify（java管程）类似的api。ConditionObject**只能在独占模式下使用**。与Object类的wait/notify相似，**Condition需要与锁一起使用**，在调用```Condition#await```和```Condition#signal```（以及对应的其它版本的api，比如带超时时间的await）之前需要先获得锁，否则会抛出```IllegalMonitorStateException```。一个锁可以与多个```ConditionObject```绑定，这是AQS比Java自带的wait/notify更加强大的地方，Java自带的api只能实现一个监视器锁对应一个condition。

调用```ConditionObject#await```的线程会被封装为Node实例添加到等待队列中，当线程被唤醒或者被中断时会被移到同步队列上。因为是通过```LockSupport.park```来实现等待时阻塞的，因此线程被唤醒可能是因为调用了```LockSupport#unpark```或者```Thread#interrupt```方法。如果线程是在被```signal```方法唤醒之前中断的话，根据ConditionObject的逻辑会抛出```InterruptedException```，而如果线程是在被```signal```方法唤醒之后被中断的话，则不会抛出异常，只是重新设置线程的中断标志为true。在这里可以先不用管具体如何实现的，在清楚的了解了AQS独占模式下工作的流程后再看对应的实现会事半功倍，因为节点从等待队列转移到同步队列以及之后在同步队列中调度的过程和独占模式下AQS中调度过程是完全相同的。

# 2.AQS的使用
当我们使用AQS构建同步工具时，需要重写以下方法：
1. 独占模式下需要重写：
    - ```protected boolean tryAcquire(int arg)```
    - ```protected boolean tryRelease(int arg)```
    - ```protected boolean isHeldExclusively()```这个方法只有需要用到Condition的时候才需要重写
2. 共享模式下需要重写：
    - ```protected int tryAcquireShared(int arg)```
    
    - ```protected boolean tryReleaseShared(int arg) ```
    

AQS源码中这几个方法默认是直接抛出异常的```throw new UnsupportedOperationException()```，没有把它们定义成abstract方法而是直接抛异常的原因可能是因为我们使用AQS时一般只会使用独占模式或者共享模式，而如果把这些方法都定义为abstract方法的话，我们在使用AQS构建同步工具的时候就需要把这几个方法都实现，所以不如直接抛异常。

# 3.AQS源码

AQS中有几个通用的方法，先在这里看一下这几个方法。

1. 内部类Node实例入队的方法```enq(final Node node) ```
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
这个方法中需要注意的是，当同步队列为空时（t == null的话，同步队列一定为空），需要先构建一个头节点，构建的这个头节点是没有封装线程的（因为成为头节点的节点是否封装线程已经没有意义了，这里可以先不用深究），并且成功构建头节点的线程并不会从该方法中返回，而是进入下一个循环执行else对应的代码，尝试把自己的node添加到同步队列中。

总结：enq(final Node node)方法提供了原子性的入队功能。

## 3.1acquire
```acquire(int arg)```为独占模式下获取资源的顶级入口，arg为获取资源的数量。
```java
public final void acquire(int arg) {
        if (!tryAcquire(arg) &&
            acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
            selfInterrupt();
    }
```
如果```tryAcquire(arg)```获取资源成功的话，会直接返回，否则的话执行```addWaiter(Node.EXCLUSIVE), arg)```。  
```addWaiter(Node.EXCLUSIVE), arg)```这个方法的作用是把线程封装为内部类Node的实例，并且把该实例添加到同步队列中，代码如下
```java
private Node addWaiter(Node mode) {
        Node node = new Node(Thread.currentThread(), mode);
        // Try the fast path of enq; backup to full enq on failure
        Node pred = tail;
        if (pred != null) {
            node.prev = pred;
            if (compareAndSetTail(pred, node)) {
                pred.next = node;
                return node;
            }
        }
        enq(node);
        return node;
    }
```

## 3.2release
## 3.3acquireShared
## 3.4releaseShared
