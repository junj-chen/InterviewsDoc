### AQS源码解析：

1. AQS的结构

~~~java
// AQS的头结点，可以表示当前持有锁的线程
private transient volatile Node head;

// 阻塞的尾结点，每一个新节点进来都插到最后， 形成一个双向链表
private transient volatile Node tail;

// 代表当前锁的状态， 0表示没有占用锁， >0表示有线程持有锁，
// 可以大于0，这个锁是可重入的锁
private volatile long state;

// 表示当前持有独占锁的线程，继承自 AbstractOwnableSynchronizer 抽象类
// reentrantLock.lock()可以嵌套调用多次，所以每次用这个来判断当前线程是否已经拥有了锁
// if (currentThread == getExclusiveOwnerThread()) {state++}
private transient Thread exclusiveOwnerThread;
~~~



AQS 结构图;

![image-20210729200812624](C:\Users\localuser\AppData\Roaming\Typora\typora-user-images\image-20210729200812624.png)



内部Node类：

~~~java
static final class Node {

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
	
    // 大于0表示该线程取消了等待
    volatile int waitStatus;
    
    // 前驱节点引用
    volatile Node prev;
    
    // 后继节点的引用
    volatile Node next;
    
    // 节点中的线程
    volatile Thread thread;
    
}
~~~

Node 的数据结构其实也挺简单的，就是 thread + waitStatus + pre + next 四个属性而已



### ReentrantLock中公平锁的实现

ReentrantLock 在内部用了内部类 Sync 来管理锁，所以真正的获取锁和释放锁是由 Sync 的实现类来控制的。

Sync 有两个实现，分别为 NonfairSync（非公平锁）和 FairSync（公平锁），我们看 FairSync 部分。

在ReentrantLock中实现加锁，调用 Sync.lock

~~~java
public void lock() {
    sync.lock();
}
~~~



~~~java


static final class FairSync extends Sync {
    private static final long serialVersionUID = -3000897897090466540L;

    final void lock() {
        acquire(1);  // 竞争锁，
    }
    
    // 父类（AQS）中的方法, 就是一个模板设计模式，
    // 定义好了模板，需要进行实现 具体的 tryAcquire(1)方法
    // tryAcquire方法返回true, 则直接结束该方法，
    // 否则会将该线程封装为一个Node压缩到 阻塞队列中
     public final void acquire(int arg) {
         // tryAcquire(1)表示尝试去获取锁，参数值 1 是因为 state默认值为0，不持有锁，
		// 设置为 1 ，表示该线程持有锁
        if (!tryAcquire(arg) &&   
            // 如果 tryAcquire(1)没有成功，就需要把当前线程挂起，放到阻塞队列中
            acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
            selfInterrupt();
    }
	
    // FairSync 公平锁的具体实现方法，尝试直接获取锁，返回值 true 为成功获取，false失败
    // 返回true情况： 1. 没有线程持有锁， 2. 改线程已经持有锁，可重入的情况
    protected final boolean tryAcquire(int acquires) {
        final Thread current = Thread.currentThread();
        int c = getState();
        // 1. c == 0 表示没有线程获取锁资源
        if (c == 0) {
            // 公平锁需要去队列中看一下有没有线程在队列中等待，
            // hasQueuedPredecessors() 查看队列是否有线程等待
            if (!hasQueuedPredecessors() &&
                // 没有线程等待，尝试CAS将获取当前线程
                // 因为可能有其他的线程同时竞争这个锁
                compareAndSetState(0, acquires)) {
                // 竞争成功，标记获取了锁，返回true
                setExclusiveOwnerThread(current);
                return true;
            }
        }
        // 2. c != 0，那就看一下当前线程是不是已经占有锁，如果占有锁，那就是重入锁
        // 对state = state + 1操作， 返回 true
        else if (current == getExclusiveOwnerThread()) {
            int nextc = c + acquires;
            if (nextc < 0)
                throw new Error("Maximum lock count exceeded");
            setState(nextc);
            return true;
        }
        // 3. 最后，没有抢到锁，也没有占有锁，返回 false
        return false;
    }
}
~~~

当线程在 tryAcquire方法中获取锁失败，那么就需要将线程加入到阻塞队列中，执行该方法

```java
acquireQueued(addWaiter(Node.EXCLUSIVE), arg)   // 首先执行addWaiter(Node.EXCLUSIVE)

// 该方法把线程包装成Node,放入阻塞队列中
	private Node addWaiter(Node mode) {
        // 当前线程封装成 Node节点 
        Node node = new Node(Thread.currentThread(), mode);
        // Try the fast path of enq; backup to full enq on failure
        // 将Node加入到阻塞队列最后面
        Node pred = tail;
        if (pred != null) {  // tail != null, 队列不为空，第一次进入，队列为空
            node.prev = pred;  // node节点的前驱指向tail节点
            if (compareAndSetTail(pred, node)) {  // 使用CAS把自己设为尾结点
                pred.next = node;  // 双向链表实现
                return node;
            }
        }
        // 上面的 if 没有执行，说明队列为空获取 CAS 竞争锁设置尾结点失败
        // 执行入队操作
        enq(node);
        return node;
    }
    
    // 采用自旋方式的入队操作
    // 执行该方法，要么队列为空，获取竞争入队失败
    // 循环自旋，CAS 自旋设置尾结点，竞争一次不到，就竞争多次
    private Node enq(final Node node) {
        for (;;) {
            Node t = tail;  // 获取尾结点
            if (t == null) { // 1. 尾结点为空，说明队列为空的情况
                if (compareAndSetHead(new Node())) // CAS 去初始化一个头结点
                 // 给后面用：这个时候head节点的waitStatus==0, 看new Node()构造方法就知道了
                    // 设置尾结点，
                    tail = head;
            } else {
                // addWaiter 中一样，设置尾结点
                node.prev = t;
                if (compareAndSetTail(t, node)) {
                    t.next = node;
                    return t;
                }
            }
        }
    }

```

上述代码（addWaiter）将 当前线程包装成Node节点，放入了队列尾部，回到 acquire 方法中，

acquireQueued(addWaiter(Node.EXCLUSIVE), arg) 返回 true,就会进入 selfInterrupt()方法，

所以正常情况下，该方法返回false

~~~java
if (!tryAcquire(arg) &&   
            // 如果 tryAcquire(1)没有成功，就需要把当前线程挂起，放到阻塞队列中
            acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
            selfInterrupt();
    }

// acquireQueued ,参数 Node 已经加入 阻塞队列了， 
// 该方法 将线程实现将线程挂起，然后被唤醒后获取锁
final boolean acquireQueued(final Node node, int arg) {
    boolean failed = true;
    try {
        boolean interrupted = false;
        for (;;) {  // 循环
            final Node p = node.predecessor(); // 获取node节点的前驱 p
            
  		 // p == head,说明当前节点虽然进入阻塞队列，但是他是第一个节点，前驱节点是head节点
            // 注意点： 阻塞队列不包含头结点，head一般指占有线程的节点，head后面的为阻塞队列
            // 所以当前节点可以去试抢一下锁
         // 这里我们说一下，为什么可以去试试：
         // 首先，它是队头，这个是第一个条件，其次，当前的head有可能是刚刚初始化的node，
         // enq(node) 方法里面有提到，head是延时初始化的，而且new Node()的时候没有设置任何线程
            if (p == head && tryAcquire(arg)) {
                setHead(node);
                p.next = null; // help GC
                failed = false;
                return interrupted;
            }
            // 上面if为 false, 要么 不是头节点， 要么 竞争锁失败
            if (shouldParkAfterFailedAcquire(p, node) &&
                parkAndCheckInterrupt())
                interrupted = true;
        }
    } finally {
        if (failed)
            cancelAcquire(node);
    }
}

// shouldParkAfterFailedAcquire 尝试获取锁失败将当前线程挂起
// 第一个参数为前驱节点， 第二个参数为 当前节点
private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
    int ws = pred.waitStatus;
     // 前驱节点的 waitStatus == -1 ，说明前驱节点状态正常，当前线程需要挂起，直接可以返回true
    if (ws == Node.SIGNAL)
        /*
             * This node has already set status asking a release
             * to signal it, so it can safely park.
             */
        return true;
    
    // 前驱节点 waitStatus大于0 ，之前说过，大于0 说明前驱节点取消了排队
    // 这里需要知道这点：进入阻塞队列排队的线程会被挂起，而唤醒的操作是由前驱节点完成的。
    // 所以下面这块代码说的是将当前节点的prev指向waitStatus<=0的节点，
    // 简单说，就是为了找个好爹，因为你还得依赖它来唤醒呢，如果前驱节点取消了排队，
    // 找前驱节点的前驱节点做爹，往前遍历总能找到一个好爹的
    if (ws > 0) {
        /*
             * Predecessor was cancelled. Skip over predecessors and
             * indicate retry.
             */
        do {
            node.prev = pred = pred.prev;
        } while (pred.waitStatus > 0);  // 找到一个小于等于0 的前驱节点，用于唤醒自己
        pred.next = node;
    } else {
        /*
             * waitStatus must be 0 or PROPAGATE.  Indicate that we
             * need a signal, but don't park yet.  Caller will need to
             * retry to make sure it cannot acquire before parking.
             */
      // 仔细想想，如果进入到这个分支意味着什么
      // 前驱节点的waitStatus不等于-1和1，那也就是只可能是0，-2，-3
   // 在我们前面的源码中，都没有看到有设置waitStatus的，所以每个新的node入队时，waitStatu都是0
     // 正常情况下，前驱节点是之前的 tail，那么它的 waitStatus 应该是 0
        
     // 用CAS将 前驱节点 的waitStatus设置为Node.SIGNAL(也就是-1)
        compareAndSetWaitStatus(pred, ws, Node.SIGNAL);
    }
    
    // 返回false，上一个函数在循环中会再执行一次
    // 进入该方法中，会从第一个 if 分支返回
    return false;
}

// 这个方法很简单，因为前面返回true，所以需要挂起线程，这个方法就是负责挂起线程的
// 这里用了LockSupport.park(this)来挂起线程，然后就停在这里了，等待被唤醒=======
private final boolean parkAndCheckInterrupt() {
    LockSupport.park(this);
    return Thread.interrupted();
}

~~~



































