[TOC]
# 1.AQS简介
首先从大局上介绍一下AQS和一些相关的知识，这部分对后面阅读源码有帮助，熟悉这些概念的同学也可以跳过。
AbstractQuenedSynchronizer从名字就能看出AQS是一个_抽象的_基于队列的同步器。AQS本身是作为框架使用的，juc（java.util.concurrent）包中很多同步工具比如ReentrantLock、CountDownLatch、Semphore、ReentrantReadWriteLock、FutureTask等类都是基于AQS来实现的，这几个工具都有作为同步工具的AQS的子类```Sync extends AbstractQueuedSynchronizer```。AQS提供了对内部资源state的原子性管理以及对线程调度的管理。
AQS内部主要有一个变量state、一个严格FIFO的CLH队列、内部类Node和ConditionObject  

1. volatile的变量state：该变量可以抽象的理解为资源的数量，比如在ReentrantLock中该变量就可以表示为是否获得锁以及获得锁的次数。

2. 先进先出（FIFO）的CLH队列：当线程尝试获取资源失败时就会加入到这个队列中，等待调度。
    Node：用于封装线程,添加了一些有用的状态标识信息，例如当前节点是否为独占模式、等待状态、前驱后去节点的引用。获取资源失败的线程会被封装为Node然后加入到CLH同步队列中。  
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
    while(pred.status == locked) { //表示前驱节点仍占用着锁,在传统的CLH队列中，这个pred并不是真正的引用，这里只是方便理解才加上了pred，实际上应
        //让出cpu                   //为status == locked，而这个status就是前驱的本地变量status，因为我们只需要在前驱的status上自旋，实际上
    }                              //并不需要前驱节点的引用
    //前驱节点使用完锁
    this.status = unlocked;
    ```
对MCS锁和CLH锁感兴趣的同学可以深入去了解一下，如果只是看AQS源码的话，知道这些就已经足够了。

3. AQS中实现的CLH队列是一个双向链表的数据结构，每个节点保存前驱节点的引用```prev```和后驱节点的引用```next```。

  [![ssLKFs.png](https://s3.ax1x.com/2021/01/17/ssLKFs.png)](https://imgchr.com/i/ssLKFs)

4. ConditionObject：用于提供条件等待队列的支持，提供类似于Object类的wait和signal的api，内部维护了一个单链表，调用了await的线程会释放已经获取的资源并加入到该链表中。当有其他线程调用signal时，被唤醒的线程（可能是多个）会重新加入CLH同步队列中参与资源的争夺。


## 1.1内部类Node
```java
static final class Node {
        /** Marker to indicate a node is waiting in shared mode */
        static final Node SHARED = new Node();
        /** Marker to indicate a node is waiting in exclusive mode */
        static final Node EXCLUSIVE = null;

        /** waitStatus value to indicate thread has cancelled */
        static final int CANCELLED =  1;
        /** waitStatus value to indicate successor's thread needs unparking */
        static final int SIGNAL    = -1;
        /** waitStatus value to indicate thread is waiting on condition */
        static final int CONDITION = -2;
        /**
         * waitStatus value to indicate the next acquireShared should
         * unconditionally propagate
         */
        static final int PROPAGATE = -3;


​        
        volatile int waitStatus;


​       
        volatile Node prev;


​      
        volatile Node next;


​       
        volatile Thread thread;


​       
        Node nextWaiter;


​       
        final boolean isShared() {
            return nextWaiter == SHARED;
        }


​      
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
## 1.2内部类ConditionObject

# 2.AQS源码
## 2.1acquire
## 2.2release
## 2.3acquireShared
## 2.4releaseShared
  ```

  ```