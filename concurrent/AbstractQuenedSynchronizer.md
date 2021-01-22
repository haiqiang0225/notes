[TOC]
# 1.AQS简介
首先从大局上介绍一下AQS和一些相关的知识，这部分对后面阅读源码有帮助，熟悉这些概念的同学也可以跳过。
AbstractQuenedSynchronizer从名字就能看出AQS是一个_抽象的_基于队列的同步器。AQS本身是作为框架使用的，juc（java.util.concurrent）包中很多同步工具比如ReentrantLock、CountDownLatch、Semphore、ReentrantReadWriteLock、FutureTask等类都是基于AQS来实现的，这几个工具都有作为同步工具的AQS的子类```Sync extends AbstractQueuedSynchronizer```。AQS提供了对内部资源state的原子性管理以及对线程调度的管理。
AQS内部主要有一个变量state、一个严格FIFO的CLH队列、内部类Node和ConditionObject，其中ConditionObject提供了等待队列，也就是说AQS中是有两个队列的：一个用于同步调度的CLH**同步队列**和一个用于实现Condition的**等待队列**。

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
    
    AQS中CLH队列实现是一个双向链表的数据结构，每个节点显式的保存前驱节点的引用```prev```和后驱节点的引用```next```，一个节点对应一个线程，只有获取资源（tryAcquire/tryAcquireShared）失败的线程，才会被封装为Node并且进入同步队列进行调度，直接获取成功的线程是不会进入同步队列进行调度的。需要注意的是head指向的节点是没有对应线程的，换句话说就是如果一个节点成为了头节点么这个节点对应的线程已经成功获得了资源了，且同步队列中只有头节点的下一个结点会尝试执行tryAcquire(Shared)来获得资源，而其他的节点是不会尝试获得资源的。

  [![ssLKFs.png](https://s3.ax1x.com/2021/01/17/ssLKFs.png)](https://imgchr.com/i/ssLKFs)

4. **ConditionObject**：用于提供条件等待队列的支持，提供类似于Object类的wait和signal的api，内部维护了一个单链表，调用了await的线程会释放已经获取的资源并加入到该链表中。当有其他线程调用signal时，被唤醒的线程（可能是多个）会重新加入CLH同步队列中参与资源的争夺。   

## 1.1内部类Node

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
         * 这个引用有两个用处，当节点位于同步队列中时，nexWaiter用于标识线程是共享
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
|waitStatus|表示了当前节点的状态，总共有5种状态。<br />  1. 0，代表初始状态或者中间状态<br />  2. SIGNAL，代表当前节点需要在释放资源后唤醒一个后继节点来获取资源<br />  3. CONDITION，代表节点此时在等待队列中而不在同步队列中<br />  4. PROPAGATE，代表下一个acquireShared应该无条件传播，在```shouldParkAfterFailedAcquire```方法中体现<br />  5. CANCELLED， 代表当前节点已经被取消，此状态是唯一一个大于0的状态|

## 1.2内部类ConditionObject

 ```java
public class ConditionObject implements Condition, java.io.Serializable {
        private static final long serialVersionUID = 1173984872572414699L;
        /** First node of condition queue. */
        private transient Node firstWaiter;
        /** Last node of condition queue. */
        private transient Node lastWaiter;

        public ConditionObject() { }

        private Node addConditionWaiter() {
        }

        private void doSignal(Node first) { 
        }

        private void doSignalAll(Node first) { 
        }

        private void unlinkCancelledWaiters() {
        }

    
        public final void signal() {
        }

        public final void signalAll() {
        }
    
        public final void awaitUninterruptibly() {
        }

      
        private static final int REINTERRUPT =  1;
       
        private static final int THROW_IE    = -1;

      
        private int checkInterruptWhileWaiting(Node node) {
        }

        private void reportInterruptAfterWait(int interruptMode)
            throws InterruptedException {
        }

      
        public final void await() throws InterruptedException {           
        }

        public final long awaitNanos(long nanosTimeout)
                throws InterruptedException {     
        }

        public final boolean awaitUntil(Date deadline)
                throws InterruptedException {         
        }
    
        public final boolean await(long time, TimeUnit unit)
                throws InterruptedException {    
        }

    
        final boolean isOwnedBy(AbstractQueuedSynchronizer sync) {
        }

        protected final boolean hasWaiters() {
        }

        protected final int getWaitQueueLength() { 
        }

        protected final Collection<Thread> getWaitingThreads() {
        }
    }
 ```
# 2.AQS源码
## 2.1acquire
## 2.2release
## 2.3acquireShared
## 2.4releaseShared
# 3.AQS的使用