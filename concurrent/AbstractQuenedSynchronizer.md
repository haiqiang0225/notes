[TOC]
# 1.AQS简介
AbstractQuenedSynchronizer从名字就能看出AQS是一个_抽象的_基于队列的同步器。AQS本身是作为框架使用的，juc中很多同步工具比如ReentrantLock、CountDownLatch、Semphore、ReentrantReadWriteLock、FutureTask等类都是基于AQS来实现的，这几个工具都有作为同步工具的AQS的子类```Sync extends AbstractQueuedSynchronizer```。AQS提供了对内部资源state的原子性管理以及对线程调度的管理。
AQS内部主要有一个变量state、一个FIFO的CLH队列、内部类Node和ConditionObject  

volatile的变量state：该变量可以理解为资源的数量，比如在ReentrantLock中该变量就可以表示为是否获得锁以及获得锁的次数。
先进先出（FIFO）的CLH队列：当线程尝试获取资源失败时就会加入到这个队列中，等待调度。
Node：用于封装线程,添加了一些有用的状态标识信息，例如当前节点是否为独占模式、等待状态、前驱后去节点的引用。获取资源失败的线程会被封装为Node然后加入到CLH同步队列中。
ConditionObject：用于提供条件等待队列的支持，提供类似于Object类的wait和signal的api，内部维护了一个单链表，调用了await的线程会释放已经获取的资源并加入到该链表中。当有其他线程调用signal时，被唤醒的线程（可能是多个）会重新加入CLH同步队列中参与资源的争夺。


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

        
        volatile int waitStatus;

       
        volatile Node prev;

      
        volatile Node next;

       
        volatile Thread thread;

       
        Node nextWaiter;

       
        final boolean isShared() {
            return nextWaiter == SHARED;
        }

      
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