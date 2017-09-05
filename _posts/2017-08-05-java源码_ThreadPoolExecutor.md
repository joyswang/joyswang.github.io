---
layout: post
title: java源码-ThreadPoolExecutor
date: 2017-08-04 16:21:00
categories:
- java
tags:
- java
- 源码
- 并发
---

我们通过源码来瞅瞅java的线程池是如何实现的。
<br/>
<br/>
<br/>
<br/>
<br/>
<br/>
<br/>

### 简单例子

先通过一个简单的例子来看看吧。

```java

public class TestThreadPoolExecutor {

	public static void main(String[] args) {
		ExecutorService executor = new ThreadPoolExecutor(1, 1,
                0L, TimeUnit.MILLISECONDS,
                new LinkedBlockingQueue<Runnable>());
		executor.execute(new Runnable() {
			
			@Override
			public void run() {
				// TODO Auto-generated method stub
				System.out.println("sssssss");
			}
		});
		executor.shutdown();
	}
}

```
* 实例化一个executor，调用execute(Runnable)方法
* 调用shutdown()方法。
* 下面我们就通过execute(Runnable)方法进入开始了解```ThreadPoolExecutor```

在了解方法前，我们先来了解下```ThreadPoolExecutor```中的一些重要的参数
* clt：高三位记录当前线程池状态，低29位记录当前线程数量
* corePoolSize：线程池核心线程数量，allowCoreThreadTimeOut(默认为false)为false且当前线程数量小于corePoolSize时，现在不会死亡。
* maximunPoolSize：线程池最大线程数量;
* keepAliveTime：线程执行玩当前任务后活着的状态; 

```java

private final AtomicInteger ctl = new AtomicInteger(ctlOf(RUNNING, 0));
//29
private static final int COUNT_BITS = Integer.SIZE - 3;
////00011111111111111111111111111111
private static final int CAPACITY   = (1 << COUNT_BITS) - 1;

// runState is stored in the high-order bits
//11100000000000000000000000000000
private static final int RUNNING    = -1 << COUNT_BITS;
//0
private static final int SHUTDOWN   =  0 << COUNT_BITS;
//00100000000000000000000000000000
private static final int STOP       =  1 << COUNT_BITS;
//01000000000000000000000000000000
private static final int TIDYING    =  2 << COUNT_BITS;
//01100000000000000000000000000000
private static final int TERMINATED =  3 << COUNT_BITS;

// Packing and unpacking ctl
//获取ctl高三位，状态
private static int runStateOf(int c)     { return c & ~CAPACITY; }
//获取线程池数量
private static int workerCountOf(int c)  { return c & CAPACITY; }
private static int ctlOf(int rs, int wc) { return rs | wc; }

```


### execute(Runnable)方法

```java

 public void execute(Runnable command) {
    if (command == null)
        throw new NullPointerException();
    /*
     * Proceed in 3 steps:
     *
     * 1. If fewer than corePoolSize threads are running, try to
     * start a new thread with the given command as its first
     * task.  The call to addWorker atomically checks runState and
     * workerCount, and so prevents false alarms that would add
     * threads when it shouldn't, by returning false.
     *
     * 2. If a task can be successfully queued, then we still need
     * to double-check whether we should have added a thread
     * (because existing ones died since last checking) or that
     * the pool shut down since entry into this method. So we
     * recheck state and if necessary roll back the enqueuing if
     * stopped, or start a new thread if there are none.
     *
     * 3. If we cannot queue task, then we try to add a new
     * thread.  If it fails, we know we are shut down or saturated
     * and so reject the task.
     */
     //得到ctl
    int c = ctl.get();
    //判断当前的worker数量是否还是小于自定义的线程池容量 (1)步
    if (workerCountOf(c) < corePoolSize) {
    	//把传入的线程添加到worker中
        if (addWorker(command, true))
            return;
        //添加不成功，得到当前的ctl（因为可能有外因导致了当前线程状态的改变，才会导致addWorker失败，所以需要重新获得ctl）
        c = ctl.get();
    }
    //当前线程池是否运行中 && 把command添加到线程队列的队尾 (2)步
    if (isRunning(c) && workQueue.offer(command)) {
        int recheck = ctl.get();
        if (! isRunning(recheck) && remove(command))
            reject(command);
        else if (workerCountOf(recheck) == 0)
        	//如果当前线程池中没有线程 调用addWorker方法
            addWorker(null, false);
    }
    // (3)步
    else if (!addWorker(command, false))
        reject(command);
}

```

* 主要分三步
	- 判断当前线程池中线程数量是否小于```corePoolSize```,小于则添加新的线程，添加成功直接返回
	- 判断当前是否运行中&向队列中添加线程，判断当前线程数量是否为0，为0则向线程池中添加线程
	- 如果向队列中添加线程失败，然后就尝试向线程池中添加一个新的线程，失败表示当前线程已经shutdown

### addWorker(Runnable, boolean)

* Runnable：线程对象
* boolean：false：表示当前线程数量不能大于```maximumPoolSize```；true：表示当前线程数量不能大于```corePoolSize```

```java

private boolean addWorker(Runnable firstTask, boolean core) {
    retry:
    for (;;) {
        int c = ctl.get();
        int rs = runStateOf(c);

        // Check if queue empty only if necessary.
        if (rs >= SHUTDOWN &&
            ! (rs == SHUTDOWN &&
               firstTask == null &&
               ! workQueue.isEmpty()))
            return false;

        for (;;) {
            int wc = workerCountOf(c);
            //当前线程数量大于CAPACITY(线程池中所能添加最大值2^29-1)
            //或者线程数量大于自定义的线程最大量，也返回false
            if (wc >= CAPACITY ||
                wc >= (core ? corePoolSize : maximumPoolSize))
                return false;
            //线程数量加一
            if (compareAndIncrementWorkerCount(c))
                break retry;
            c = ctl.get();  // Re-read ctl
            //线程数量新增失败，如果当前线程状态已经改变，则需要重新判断当前线程的状态
            //如果线程状态没变，则说明线程数量变了，只需要重新新增线程数量即可
            if (runStateOf(c) != rs)
                continue retry;
            // else CAS failed due to workerCount change; retry inner loop
        }
    }

    boolean workerStarted = false;
    boolean workerAdded = false;
    Worker w = null;
    try {
    	// 封装到worker中
        w = new Worker(firstTask);
        final Thread t = w.thread;
        if (t != null) {
            final ReentrantLock mainLock = this.mainLock;
            //获取锁
            mainLock.lock();
            try {
                // Recheck while holding lock.
                // Back out on ThreadFactory failure or if
                // shut down before lock acquired.
                int rs = runStateOf(ctl.get());
                //判断线程池当前是否是RUNNING状态
                //亦或线程池状态是SHUTDOWN&传入的线程是null
                if (rs < SHUTDOWN ||
                    (rs == SHUTDOWN && firstTask == null)) {
                    //判断当前线程是否已经运行
                    if (t.isAlive()) // precheck that t is startable
                        throw new IllegalThreadStateException();
                    //把worker添加到workers中
                    workers.add(w);
                    int s = workers.size();
                    //设置largestPoolSize
                    if (s > largestPoolSize)
                        largestPoolSize = s;
                    workerAdded = true;
                }
            } finally {
                mainLock.unlock();
            }
            if (workerAdded) {
            	//运行线程
                t.start();
                workerStarted = true;
            }
        }
    } finally {
        if (! workerStarted)
            addWorkerFailed(w);
    }
    return workerStarted;
}

```

* 先根据线程池的状态，把线程池数量```workercount```数量+1
* 把线程封装到```Worker```类中，然后添加到```workers```集合中
* 如果添加成功，则运行线程
* 看到这里注意到，这个运行的线程是```worker.thread```，那跟我们传入的```firstTask```有什么关系呢

### Worker

```java

private final class Worker
    extends AbstractQueuedSynchronizer
    implements Runnable
{
    /**
     * This class will never be serialized, but we provide a
     * serialVersionUID to suppress a javac warning.
     */
    private static final long serialVersionUID = 6138294804551838833L;

    /** Thread this worker is running in.  Null if factory fails. */
    final Thread thread;
    /** Initial task to run.  Possibly null. */
    Runnable firstTask;
    /** Per-thread task counter */
    volatile long completedTasks;

    /**
     * Creates with given first task and thread from ThreadFactory.
     * @param firstTask the first task (null if none)
     */
    Worker(Runnable firstTask) {
        setState(-1); // inhibit interrupts until runWorker
        this.firstTask = firstTask;
        this.thread = getThreadFactory().newThread(this);
    }

    /** Delegates main run loop to outer runWorker  */
    public void run() {
        runWorker(this);
    }

    // Lock methods
    //
    // The value 0 represents the unlocked state.
    // The value 1 represents the locked state.

    protected boolean isHeldExclusively() {
        return getState() != 0;
    }

    protected boolean tryAcquire(int unused) {
        if (compareAndSetState(0, 1)) {
            setExclusiveOwnerThread(Thread.currentThread());
            return true;
        }
        return false;
    }

    protected boolean tryRelease(int unused) {
        setExclusiveOwnerThread(null);
        setState(0);
        return true;
    }

    public void lock()        { acquire(1); }
    public boolean tryLock()  { return tryAcquire(1); }
    public void unlock()      { release(1); }
    public boolean isLocked() { return isHeldExclusively(); }

    void interruptIfStarted() {
        Thread t;
        if (getState() >= 0 && (t = thread) != null && !t.isInterrupted()) {
            try {
                t.interrupt();
            } catch (SecurityException ignore) {
            }
        }
    }
}

```
* 很明显可以看到，worker.thread是线程工厂生成的，传入的是worker本身这个Runnable对象
* 所以```t.start()```运行的是worker中的run()方法，也就是```runWorker(this)```方法

### runWorker(this)

```java

final void runWorker(Worker w) {
    Thread wt = Thread.currentThread();
    Runnable task = w.firstTask;
    w.firstTask = null;	//	辅助gc
    w.unlock(); // allow interrupts
    boolean completedAbruptly = true;
    try {
    	// 如果线程task不为空，说明是新增加的线程
    	// 当然task为空不代表，这个线程就一定不是新添加的线程
    	// 如果task为空，则通过getTask()方法尝试获取task
        while (task != null || (task = getTask()) != null) {
            w.lock();
            // If pool is stopping, ensure thread is interrupted;
            // if not, ensure thread is not interrupted.  This
            // requires a recheck in second case to deal with
            // shutdownNow race while clearing interrupt
            // 如果线程正在停止或已经停止，则interrupt当前线程
            if ((runStateAtLeast(ctl.get(), STOP) ||
                 (Thread.interrupted() &&
                  runStateAtLeast(ctl.get(), STOP))) &&
                !wt.isInterrupted())
                wt.interrupt();
            try {
                beforeExecute(wt, task);
                Throwable thrown = null;
                try {
                	// 运行线程，也就是firstTask，也就是传入的线程
                    task.run();
                } catch (RuntimeException x) {
                    thrown = x; throw x;
                } catch (Error x) {
                    thrown = x; throw x;
                } catch (Throwable x) {
                    thrown = x; throw new Error(x);
                } finally {
                    afterExecute(task, thrown);
                }
            } finally {
                task = null;
                w.completedTasks++;
                w.unlock();
            }
        }
        completedAbruptly = false;
    } finally {
        processWorkerExit(w, completedAbruptly);
    }
}

```

* 如果传入的```Worker```的firstTask不为空，则直接运行这个task
* 如果为空，则通过```getTask()```方法获取task。
* 然后里面会有两个钩子方法```beforeExecute```和```afterExecute```方法

### getTask()

```java

private Runnable getTask() {
    boolean timedOut = false; // Did the last poll() time out?

    for (;;) {
        int c = ctl.get();
        int rs = runStateOf(c);

        // Check if queue empty only if necessary.
        //判断当前线程是否shutdown
        if (rs >= SHUTDOWN && (rs >= STOP || workQueue.isEmpty())) {
            decrementWorkerCount();
            return null;
        }

        int wc = workerCountOf(c);

        // Are workers subject to culling?
        // 判断从队列中获取task时使用poll还是take
        boolean timed = allowCoreThreadTimeOut || wc > corePoolSize;
        // 线程数量大于maximumPoolSize || 已经获取超时过
        // 线程队列为空
        if ((wc > maximumPoolSize || (timed && timedOut))
            && (wc > 1 || workQueue.isEmpty())) {
            // 线程数量减一，返回null，关闭当前线程
            if (compareAndDecrementWorkerCount(c))
                return null;
            continue;
        }

        try {
        	//获取线程
            Runnable r = timed ?
                workQueue.poll(keepAliveTime, TimeUnit.NANOSECONDS) :
                workQueue.take();
            if (r != null)
                return r;
            timedOut = true;
        } catch (InterruptedException retry) {
            timedOut = false;
        }
    }
}

```

* 这个方法的作用其实就是从线程队列中获取线程。

### 总结

* 看到这里，线程池的总体流程就出来了。
1. 通过入口execute调用方法
2. 判断当前线程池中的总线程数量是否小于```corePoolSize```，小于则直接添加一个线程worker1；大于则把runnable添加到队列中，如果添加到队列失败，则会想线程池中添加非核心线程，但是线程数量不能大于```maximumPoolSize```和```CAPACITY```
3. 如果worker1运行完firstTask，则会继续从线程池中获取task。获取线程时，使用take还是poll根据```allowCoreThreadTimeOut```和```wc > corePoolSize```来确定，超时时间根据```keepAliveTime```来确定
4. 如果线程队列为空，并且已经超时，则关闭当前worker1。如果```timed```为false的话，则worker1不会关闭。

### Executors.newCachedThreadPool()

```java

public static ExecutorService newCachedThreadPool() {
    return new ThreadPoolExecutor(0, Integer.MAX_VALUE,
                                  60L, TimeUnit.SECONDS,
                                  new SynchronousQueue<Runnable>());
}
public ThreadPoolExecutor(int corePoolSize,
                          int maximumPoolSize,
                          long keepAliveTime,
                          TimeUnit unit,
                          BlockingQueue<Runnable> workQueue) {
    this(corePoolSize, maximumPoolSize, keepAliveTime, unit, workQueue,
         Executors.defaultThreadFactory(), defaultHandler);
}
```

* corePoolSize:0 核心线程为0，表示当线程队列为空时，所有的线程都会关闭
* maximumPoolSize:Integer.MAX_VALUE 最大的线程数量，其实```CAPACITY```比这个还小
* keepAliveTime:60 线程从队列中获取task的超时时间，超时后关闭当前线程
* 线程队列采用```SynchronousQueue```
	- SynchronousQueue有个特点：没有数据缓冲，生产者线程对其的插入操作put必须等待消费者的移除操作take。
	- 所以第一次execute(Runnable)调用时，会运行到(3)步；如果运行完firstTask，在60秒内又调用execute(Runnable)，则会运行(2)步

### Executors.newFixedThreadPool(int nThreads)

```java

public static ExecutorService newFixedThreadPool(int nThreads) {
    return new ThreadPoolExecutor(nThreads, nThreads,
                                  0L, TimeUnit.MILLISECONDS,
                                  new LinkedBlockingQueue<Runnable>());
}


```

* corePoolSize：nThreads，核心线程数为设置值
* maximumPoolSize：nThreads，最大线程数也为设置值，maximumPoolSize与corePoolSize一样大，所以```timed```永远都是false，除非主动的设置```allowCoreThreadTimeOut```的值，所以getTask()时队列获取task时采用的是take()，获取不到会一直阻塞；而且线程也不会自动关闭。
* keepAliveTime：0，
* 线程队列采用```LinkedBlockingQueue```

