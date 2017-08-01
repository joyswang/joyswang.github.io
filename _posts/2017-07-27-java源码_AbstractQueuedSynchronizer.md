---
title: java源码--AbstractQueuedSynchronizer实现一个独占锁
date: 2017-07-27 23:48:00
categories:
- java
tags:
- 并发
- java
- 源码
---

这里我们要讲AQS是如何去实现一个独占锁的

### 通过AQS实现一个简单的Lock

java并发包下有个非常重要的类```AbstractQueuedSynchronizer```，```ReentrantLock```、```ReentrantReadWriteLock```、```CountDownLatch```、```Semaphore```都是通过```AbstractQueuedSynchronizer```（以下简称AQS）来实现的；既然AQS的用处这么大，那么这里就让我们通过阅读AQS的源码来了解一下它到底提供了一个什么样的功能。

AQS的源码中有下面的一段注释的代码，

```java

class Mutex implements Lock, java.io.Serializable {

   // Our internal helper class
   private static class Sync extends AbstractQueuedSynchronizer {
     // Reports whether in locked state
     protected boolean isHeldExclusively() {
       return getState() == 1;
     }

     // Acquires the lock if state is zero
     public boolean tryAcquire(int acquires) {
       assert acquires == 1; // Otherwise unused
       if (compareAndSetState(0, 1)) {
         setExclusiveOwnerThread(Thread.currentThread());
         return true;
       }
       return false;
     }

     // Releases the lock by setting state to zero
     protected boolean tryRelease(int releases) {
       assert releases == 1; // Otherwise unused
       if (getState() == 0) throw new IllegalMonitorStateException();
       setExclusiveOwnerThread(null);
       setState(0);
       return true;
     }

     // Provides a Condition
     Condition newCondition() { return new ConditionObject(); }

     // Deserializes properly
     private void readObject(ObjectInputStream s)
         throws IOException, ClassNotFoundException {
       s.defaultReadObject();
       setState(0); // reset to unlocked state
     }
   }

   // The sync object does all the hard work. We just forward to it.
   private final Sync sync = new Sync();

   public void lock()                { sync.acquire(1); }
   public boolean tryLock()          { return sync.tryAcquire(1); }
   public void unlock()              { sync.release(1); }
   public Condition newCondition()   { return sync.newCondition(); }
   public boolean isLocked()         { return sync.isHeldExclusively(); }
   public boolean hasQueuedThreads() { return sync.hasQueuedThreads(); }
   public void lockInterruptibly() throws InterruptedException {
     sync.acquireInterruptibly(1);
   }
   public boolean tryLock(long timeout, TimeUnit unit)
       throws InterruptedException {
     return sync.tryAcquireNanos(1, unit.toNanos(timeout));
   }
 }}

```

* 这段代码的意思就是通过AQS来简单的实现一个Lock，这个锁只有两个状态0、1，0：已释放锁，1：获取锁，并且这个锁是不可重入的。
* 从这个例子上来看，实现一个这样一个锁，主要依赖的是AQS的```acquire```和```release```方法。
* 那么下面我们就通过```acquire```和```release```方法这两个方法去认识AQS到底做了些啥

### AQS的acquire方法

```java

public final void acquire(int arg) {
	//Node.EXCLUSIVE代表独占锁
    if (!tryAcquire(arg) &&
        acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
        selfInterrupt();
}

```

* 调用```tryAcquire(int)```方法，成功则方法执行成功，返回void
* ```tryAcquire(int)```尝试获取锁失败，则调用```acquireQueued```方法。
* ```acquireQueued```方法内会先调用```addWaiter```方法，那么我们先看看```addWaiter```方法

#### addWaiter

```java

private Node addWaiter(Node mode) {
	//根据当前线程new一个新的node节点
    Node node = new Node(Thread.currentThread(), mode);
    // Try the fast path of enq; backup to full enq on failure
    //获取尾节点
    Node pred = tail;
    if (pred != null) {
    	//设置当前节点的prev为原尾节点
        node.prev = pred;
        //通过cas方法把当前节点设置为尾节点，并把原尾节点的next设置为当前节点
        if (compareAndSetTail(pred, node)) {
            pred.next = node;
            return node;	//放回当前节点
        }
    }
    enq(node);
    return node;
}
//尾节点为空
private Node enq(final Node node) {
    for (;;) {
        Node t = tail;
        //重新判断当前尾节点是否为空
        //为空表示当前链表为空
        if (t == null) { // Must initialize
        	//通过cas设置当前head节点，并把tail赋值为head
            if (compareAndSetHead(new Node()))
                tail = head;
        } else {
            node.prev = t;
            if (compareAndSetTail(t, node)) {
                t.next = node;
                return t;
            }
        }
    }
}

```

* 根据当前线程初始化一个node节点
* 如果当前链表的尾节点为null，表面当前链接还未初始化，那么初始化链接，设置tail=head=new node()
* 如果当前链表的尾节点不为null，则把当前节点放到链表的末尾
* 替换tail和head值都是通过cas操作来保证线程安全

#### acquireQueued

```java

private final boolean acquireQueued(final Node node, int arg) {
    boolean failed = true;
    try {
    	//线程是否interrupted
        boolean interrupted = false;
        for (;;) {
        	//当前节点的上一个节点
            final Node p = node.predecessor();
            //如果当前节点是头节点，并且尝试获取锁
            if (p == head && tryAcquire(arg)) {
            	//获取锁成功，设置头节点。
                setHead(node);
                p.next = null; // help GC
                failed = false;
                return interrupted;
            }
            if (shouldParkAfterFailedAcquire(p, node) &&
                parkAndCheckInterrupt())
                interrupted = true;
        }
    } finally {
        if (failed)
            cancelAcquire(node);
    }
}

//获取锁失败后，当前线程需要阻塞等待
private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
    int ws = pred.waitStatus;
    //判断上一个节点的状态是否是 SIGNAL
    if (ws == Node.SIGNAL)
        /*
         * This node has already set status asking a release
         * to signal it, so it can safely park.
         */
        return true;
    if (ws > 0) {
    	//	表示当前正在处理的线程已经cancelled
        /*
         * Predecessor was cancelled. Skip over predecessors and
         * indicate retry.
         */
        do {
            node.prev = pred = pred.prev;
        } while (pred.waitStatus > 0);
        pred.next = node;
    } else {
    	//node的waitStatus 初始状态都是 0，这里需要替换成SIGNAL，表示后面有节点正在等待唤醒
        /*
         * waitStatus must be 0 or PROPAGATE.  Indicate that we
         * need a signal, but don't park yet.  Caller will need to
         * retry to make sure it cannot acquire before parking.
         */
        compareAndSetWaitStatus(pred, ws, Node.SIGNAL);
    }
    return false;
}

//阻塞当前线程，并且当前线程是否interrupted
private final boolean parkAndCheckInterrupt() {
    LockSupport.park(this);
    return Thread.interrupted();
}

```

* 首先判断上一个节点是否是head节点，如果是，则尝试获取锁，获取成功则直接返回。
* 尝试获取锁失败，则去判断prev节点的状态。
* SIGNAL：阻塞当前线程，等待唤醒。0：表示初始状态。>0：表示已取消
* 然后一直循环直到获取锁成功。

#### cancelAcquire方法

当获取锁失败时，在finally里面会调用cancelAcquire(Node node)方法。

```java

 private void cancelAcquire(Node node) {
    // Ignore if node doesn't exist
    if (node == null)
        return;

    node.thread = null;

    // Skip cancelled predecessors
    //获取当前节点的prev节点，并跳过waitStatus=CANCELLED的节点
    Node pred = node.prev;
    while (pred.waitStatus > 0)
        node.prev = pred = pred.prev;

    // predNext is the apparent node to unsplice. CASes below will
    // fail if not, in which case, we lost race vs another cancel
    // or signal, so no further action is necessary.
    Node predNext = pred.next;

    // Can use unconditional write instead of CAS here.
    // After this atomic step, other Nodes can skip past us.
    // Before, we are free of interference from other threads.
    //设置当前节点为CANCELLED
    node.waitStatus = Node.CANCELLED;

    // If we are the tail, remove ourselves.
    // 如果是当前节点是tail节点，把pred节点设置为tail节点，并且pred节点的next节点为空
    if (node == tail && compareAndSetTail(node, pred)) {
        compareAndSetNext(pred, predNext, null);
    } else {
        // If successor needs signal, try to set pred's next-link
        // so it will get one. Otherwise wake it up to propagate.
        int ws;
        if (pred != head &&
            ((ws = pred.waitStatus) == Node.SIGNAL ||
             (ws <= 0 && compareAndSetWaitStatus(pred, ws, Node.SIGNAL))) &&
            pred.thread != null) {
            Node next = node.next;
            if (next != null && next.waitStatus <= 0)
            //pred节点不是head节点
            //pred节点的状态是 SIGNAL
            //pred的thread不为空
            //next节点不为空并且next节点状态不为CANCELLED
            //则替换pred的next节点为当前节点的next节点
                compareAndSetNext(pred, predNext, next);
        } else {
        	//直接尝试唤醒  实际上唤醒的是从链表末尾的第一个 waitStatus<=0的节点
            unparkSuccessor(node);
        }

        node.next = node; // help GC
    }
}

```

* 主要的作用就是获取锁失败后删除当前添加的节点。

上面我们了解了acquire方法后，发现内部是通过一个链表来实现功能的，所以这里我们先来了解一下这个链表的Node

### AQS的链表节点Node 

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

    /**
     * Status field, taking on only the values:
     *   SIGNAL:     The successor of this node is (or will soon be)
     *               blocked (via park), so the current node must
     *               unpark its successor when it releases or
     *               cancels. To avoid races, acquire methods must
     *               first indicate they need a signal,
     *               then retry the atomic acquire, and then,
     *               on failure, block.
     *   CANCELLED:  This node is cancelled due to timeout or interrupt.
     *               Nodes never leave this state. In particular,
     *               a thread with cancelled node never again blocks.
     *   CONDITION:  This node is currently on a condition queue.
     *               It will not be used as a sync queue node
     *               until transferred, at which time the status
     *               will be set to 0. (Use of this value here has
     *               nothing to do with the other uses of the
     *               field, but simplifies mechanics.)
     *   PROPAGATE:  A releaseShared should be propagated to other
     *               nodes. This is set (for head node only) in
     *               doReleaseShared to ensure propagation
     *               continues, even if other operations have
     *               since intervened.
     *   0:          None of the above
     *
     * The values are arranged numerically to simplify use.
     * Non-negative values mean that a node doesn't need to
     * signal. So, most code doesn't need to check for particular
     * values, just for sign.
     *
     * The field is initialized to 0 for normal sync nodes, and
     * CONDITION for condition nodes.  It is modified using CAS
     * (or when possible, unconditional volatile writes).
     */
    volatile int waitStatus;

    /**
     * Link to predecessor node that current node/thread relies on
     * for checking waitStatus. Assigned during enqueuing, and nulled
     * out (for sake of GC) only upon dequeuing.  Also, upon
     * cancellation of a predecessor, we short-circuit while
     * finding a non-cancelled one, which will always exist
     * because the head node is never cancelled: A node becomes
     * head only as a result of successful acquire. A
     * cancelled thread never succeeds in acquiring, and a thread only
     * cancels itself, not any other node.
     */
    volatile Node prev;

    /**
     * Link to the successor node that the current node/thread
     * unparks upon release. Assigned during enqueuing, adjusted
     * when bypassing cancelled predecessors, and nulled out (for
     * sake of GC) when dequeued.  The enq operation does not
     * assign next field of a predecessor until after attachment,
     * so seeing a null next field does not necessarily mean that
     * node is at end of queue. However, if a next field appears
     * to be null, we can scan prev's from the tail to
     * double-check.  The next field of cancelled nodes is set to
     * point to the node itself instead of null, to make life
     * easier for isOnSyncQueue.
     */
    volatile Node next;

    /**
     * The thread that enqueued this node.  Initialized on
     * construction and nulled out after use.
     */
    volatile Thread thread;

    /**
     * Link to next node waiting on condition, or the special
     * value SHARED.  Because condition queues are accessed only
     * when holding in exclusive mode, we just need a simple
     * linked queue to hold nodes while they are waiting on
     * conditions. They are then transferred to the queue to
     * re-acquire. And because conditions can only be exclusive,
     * we save a field by using special value to indicate shared
     * mode.
     */
    Node nextWaiter;

    /**
     * Returns true if node is waiting in shared mode.
     */
    final boolean isShared() {
        return nextWaiter == SHARED;
    }

    /**
     * Returns previous node, or throws NullPointerException if null.
     * Use when predecessor cannot be null.  The null check could
     * be elided, but is present to help the VM.
     *
     * @return the predecessor of this node
     */
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

| waitStatus | 说明 |
| :----: | :----: |
|   SIGNAL  | 表示当前节点的next节点正在等待唤醒 |
|   CANCELLED  | 表示当前节点已取消获取锁 |
|   CONDITION  | 表示当前节点在condition队列上 |
|   PROPAGATE  |  |
|   0  | 一个非codition的Node节点初始状态 |

* nextWaiter ：SHARED 表示当前节点是共享锁，EXCLUSIVE 表示当前是独占锁

### AQS的release方法

```java

public final boolean release(int arg) {
	//尝试释放锁
    if (tryRelease(arg)) {
        Node h = head;
        //head节点waitStatus=0 表示后面没有其他节点
        if (h != null && h.waitStatus != 0)
        	//唤醒下一个节点
            unparkSuccessor(h);
        return true;
    }
    return false;
}

/**
 * Wakes up node's successor, if one exists.
 *
 * @param node the node
 */
private void unparkSuccessor(Node node) {
    /*
     * If status is negative (i.e., possibly needing signal) try
     * to clear in anticipation of signalling.  It is OK if this
     * fails or if status is changed by waiting thread.
     */
    int ws = node.waitStatus;
    //重置waitStatus为0
    if (ws < 0)
        compareAndSetWaitStatus(node, ws, 0);

    /*
     * Thread to unpark is held in successor, which is normally
     * just the next node.  But if cancelled or apparently null,
     * traverse backwards from tail to find the actual
     * non-cancelled successor.
     */
    //获取next节点
    Node s = node.next;
    //如果next节点为空或者next节点为取消状态
    //则从后往前遍历链表获取非node节点并且 waitStatus<=0的那个节点
    if (s == null || s.waitStatus > 0) {
        s = null;
        for (Node t = tail; t != null && t != node; t = t.prev)
            if (t.waitStatus <= 0)
                s = t;
    }
    //直接唤醒
    if (s != null)
        LockSupport.unpark(s.thread);
}

```

* 尝试释放锁，成功则根据当前head节点去唤醒下一个节点
* 唤醒过程中会先把当前节点的waitStatus设置为0,这边设置为0后，acquire时便不会去park阻塞next节点了。
* 失败则直接返回false


### AQS的acquireInterruptibly

```java

public final void acquireInterruptibly(int arg)
        throws InterruptedException {
    if (Thread.interrupted())
        throw new InterruptedException();
    if (!tryAcquire(arg))
        doAcquireInterruptibly(arg);
}

private void doAcquireInterruptibly(int arg)
    throws InterruptedException {
    final Node node = addWaiter(Node.EXCLUSIVE);
    boolean failed = true;
    try {
        for (;;) {
            final Node p = node.predecessor();
            if (p == head && tryAcquire(arg)) {
                setHead(node);
                p.next = null; // help GC
                failed = false;
                return;
            }
            if (shouldParkAfterFailedAcquire(p, node) &&
                parkAndCheckInterrupt())
                throw new InterruptedException();
        }
    } finally {
        if (failed)
            cancelAcquire(node);
    }
}

```

* 如果仔细的看完过上面的acquire方法的同学，这里应该就能一目了然了。
* 这个方法跟acquire方法不同的地方在于，如果线程已经中断，则直接抛出InterruptedException。




