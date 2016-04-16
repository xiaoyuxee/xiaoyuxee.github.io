---
title: Java中的多线程（五）：ThreadPoolExecutor框架源码解析
date: 2016-04-07 17:07:43
tags: 
- Concurrence
- ThreadPool
categories: Java-Core
toc: true
---

启动线程任务最简单的方式是：为每个任务都创建一个线程（per-task）。在没有超出服务器处理能力时，这种方法既可以提升响应速度、又可以提升吞吐量。

而在实际应用中，这种*thread-per-task*存在缺陷，特别是需要创建大量的线程时：

* 线程创建的开销：线程的创建与关闭不是“免费”的
* 资源消耗量：活动的线程会消耗系统资源

所以，在一定范围内，增加线程可以提高系统的吞吐量。但一旦超过这个范围，再创建更多的线程只会增加系统开销，并有可能导致应用程序崩溃。所以应该限制你的应用程序可以创建的线程数量，确保线程数达到这个极限时，程序也不至于耗尽所有资源。于是线程池应运而生。

<!-- more -->

## ThreadPoolExecutor
Java中内置的线程池，主要通过`Executors`的静态工厂方法创建不同类型的线程池，如：
``` java
public class Executors {

    public static ExecutorService newSingleThreadExecutor() {}

    public static ExecutorService newFixedThreadPool(int nThreads) {}

    public static ExecutorService newCachedThreadPool() {}
}
```

## 相关类设计
### Executor
用于提交任务的接口：将任务的提交与任务执行细节隔离。
``` java
public interface Executor {

    void execute(Runnable command);
}
```

### ExecutorService
线程状态/生命周期的管理接口：提交任务、关闭线程池、查询线程池状态等。
``` java
public interface ExecutorService extends Executor {
    
    void shutdown();

    List<Runnable> shutdownNow();

    boolean isShutdown();

    boolean isTerminated();

    <T> Future<T> submit(Callable<T> task);

    <T> Future<T> submit(Runnable task, T result);
}
```

## 基本结构
* corePoolSize：最小的活跃线程数
* maximumPoolSize：最大的处理作业线程数，受限于`CAPACITY`
* workers：线程池中所有的工作线程
* workQueue：阻塞（缓存）队列，向工作的线程传递任务
* ctl：线程池状态(AtomicInteger)，按二进制的位数切分后包含两部分：`runState|workerCount`
  * workerCount：活跃的线程数，范围(2^29)-1，占据ctl的前28位
  * runState：线程池状态包括运行、正在关闭等，由剩下的4位组成
    * RUNNING：接受新的任务，并且处理队列中的任务
    * SHUTDOWN：不再接受新的任务，但是会处理完队列中任务
    * STOP：不再接受新的任务，也不会处理队列中的任务，同时会中断正在执行的任务
    * TIDYING：所有任务被终止，workerCount=0。会调用`terminated()`方法
    * TERMINATED：`terminated()`方法执行完毕
    以上这些状态是可比较的，依次递增

## 线程池中主要原理/机制

### 任务的执行机制
线程的执行机制主要在`execute`方法中，任务的提交其实是封装线程后，再执行：
``` java
public <T> Future<T> submit(Runnable task, T result) {
    if (task == null) throw new NullPointerException();
    // 构造Future
    RunnableFuture<T> ftask = newTaskFor(task, result);

    // 执行
    execute(ftask);
    return ftask;
}
```
当一个线程被提交后，线程池的工作机制为：

1. 如果当前工作的线程数小于设置的核心线程数，则新建一个线程执行此任务
2. 如果工作的线程数已满足核心线程数，则将任务放入用于存放待执行任务的队列中（后续会被处理）
3. 如果缓存队列已满，则新建一个线程执行此任务（此时工作线程数*大于*核心线程数）

``` java
public void execute(Runnable command) {
    if (command == null)
        throw new NullPointerException();

    int c = ctl.get();
    if (workerCountOf(c) < corePoolSize) { // 如果工作线程小于核心线程池数量
        if (addWorker(command, true)) // 创建一个新的核心线程来处理当前任务
            return;
        c = ctl.get();
    }
    if (isRunning(c) && workQueue.offer(command)) { // 如果线程池还在运行中，将任务添加到缓存队列
        int recheck = ctl.get(); // 添加到缓存队列后，仍然需要二次检查
        if (! isRunning(recheck) && remove(command)) // 如果线程池关闭，删除刚刚添加的任务
            reject(command); // 如果成功删除，则执行任务拒绝流程
        else if (workerCountOf(recheck) == 0) // 如果没有在工作的线程，需要启动一个线程来队列中任务
            addWorker(null, false); // 创建一个线程来处理刚刚被加入到队列的任务
    }
    else if (!addWorker(command, false)) // 如果缓存队列已满，创建一个新的线程来处理当前任务
        reject(command);
}
```
#### 状态二次校验
将任务放入缓存队列后有一个二次校验：如果线程池已经关闭，那么将其从队列中剔除，并且直接执行拒绝流程。当然没有这块判断，这个任务最终也不会被执行，只是反馈被延迟了。

### 线程的新增机制
`addWorker`方法至关重要，因为增加工作线程总是伴随着任务的提交，新建的线程需要立刻用于执行被提交的任务：

1. 首先检查线程池状态：排除明显不需要再创建线程的场景，如线程池已经关闭则不再接受新的任务，或者正在关闭时，队列中的任务已经为空
2. 然后检查线程上限，然后增加线程池中工作线程的数量
3. 最后获取线程池锁，新增工作线程，*并将其启动*

``` java
// 参数core在判断工作线程上限时使用:
// true:表示上限为核心线程数, false:表示上限为最大线程数.
private boolean addWorker(Runnable firstTask, boolean core) {
    retry:
    for (;;) {
        int c = ctl.get();
        int rs = runStateOf(c);

        // Check if queue empty only if necessary.
        // 线程池非运行状态下：
        // 1. 非正在关闭状态：不再增加工作线程，因为已经在强制关闭或已经关闭
        // 2. task!=null：不再接受新的任务，但可能会创建新的线程来处理队列中任务
        // 3. 缓存队列为空：不需要线程线程，因为没有需要处理的任务了
        if (rs >= SHUTDOWN &&
            ! (rs == SHUTDOWN &&
               firstTask == null &&
               ! workQueue.isEmpty()))
            return false;

        // 更新工作线程数量：
        // 1. 更新机制：CAS(CompareAndSet)，确保是当前线程执行了更新
        // 2. 如果线程池运行状态发生变更，需要重新获取线程池状态：ctl
        for (;;) {
            int wc = workerCountOf(c);

            // 工作线程数量的上限判断
            if (wc >= CAPACITY ||
                wc >= (core ? corePoolSize : maximumPoolSize))
                return false;

            // 如果工作线程数量“更加”成功（n=>n+1），则跳出retry，去创建线程
            if (compareAndIncrementWorkerCount(c))
                break retry;

            c = ctl.get();  // Re-read ctl
            if (runStateOf(c) != rs)
                continue retry; 线程池运行状态发生变更，需要重新获取线程池状态：ctl
            // else CAS failed due to workerCount change; retry inner loop
        }
    }

    boolean workerStarted = false;
    boolean workerAdded = false;
    Worker w = null;
    try {
        w = new Worker(firstTask);
        final Thread t = w.thread;
        if (t != null) {
            final ReentrantLock mainLock = this.mainLock;
            mainLock.lock();
            try {
                // Recheck while holding lock.
                // Back out on ThreadFactory failure or if
                // shut down before lock acquired.
                int rs = runStateOf(ctl.get());

                // 需要创建新线程的场景：
                // 1. 运行状态
                // 2. 正在关闭并且该线程是用来处理缓存队列中任务(task=null) 
                if (rs < SHUTDOWN ||
                    (rs == SHUTDOWN && firstTask == null)) {
                    if (t.isAlive()) // precheck that t is startable
                        throw new IllegalThreadStateException();
                    workers.add(w);
                    int s = workers.size();
                    if (s > largestPoolSize)
                        largestPoolSize = s;
                    workerAdded = true;
                }
            } finally {
                mainLock.unlock();
            }
            if (workerAdded) {
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

#### 状态变量的变化没有上锁
1. 因为线程池的状态变量已为线程安全的，也就没有通过外部锁去控制它变化的原子性，这样将有很大的灵活性，多个线程在检查线程池状态时不存在竞争关系。但这也要求：在修改状态变量ctl时必须保证同时只有一个线程去修改它，因为它的检查与修改不具有原子性。这里用的是jvm底层`CAS-check and set`机制
2. 工作线程也没有上锁，所以工作线程变更时需要获取外部锁

### 工作线程worker的运行机制
内部类`Worker`实现了`Runnable`，也是一个任务，run方法委托给了外部的`runWorker`。

* 工作线程被启动后总是先运行第一个任务（也就是初始化工作线程时的任务）
* 如果第一个任务任务为`null`，则*循环*从缓存缓存队列中获取任务，直到队列为空
* 每次获取任务后，要检查线程池状态，必要时（线程池已关闭）及时中断该工作线程
* 缓存队列为空后，执行线程组退出流程：统计已完成的任务数量等

``` java
private final class Worker extends AbstractQueuedSynchronizer implements Runnable {
    public void run() {
        runWorker(this);
    }
}

final void runWorker(Worker w) {
    Thread wt = Thread.currentThread();
    Runnable task = w.firstTask;
    w.firstTask = null;
    w.unlock(); // allow interrupts
    // 防止工作线程处理队列中任务时发生异常，预设标记值
    boolean completedAbruptly = true;
    try {
        // 循环获取任务
        while (task != null || (task = getTask()) != null) {
            w.lock();
            // If pool is stopping, ensure thread is interrupted;
            // if not, ensure thread is not interrupted.  This
            // requires a recheck in second case to deal with
            // shutdownNow race while clearing interrupt
            // 每次执行任务时，都需要检查线程池状态，及时中断当前线程
            if ((runStateAtLeast(ctl.get(), STOP) ||
                 (Thread.interrupted() &&
                  runStateAtLeast(ctl.get(), STOP))) &&
                !wt.isInterrupted())
                wt.interrupt();
            try {
                beforeExecute(wt, task);
                Throwable thrown = null;
                try {
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
        completedAbruptly = false; // 改工作线程工作正常，没有发生异常
    } finally {
        // 缓存队列中已经没有任务，则可以退出了：修改线程池相关状态
        processWorkerExit(w, completedAbruptly);
    }
}
```

### 获取缓存队列中任务
1. 检查线程池状态：如果线程池正在关闭却队列已经为空，或者线程池已关闭，修改工作线程数量，因为当前线程会自然运行结束
2. 维护核心线程数量：超过核心线程后需要停掉该线程（此时已经完成工作线程的第一个任务）

``` java
private Runnable getTask() {
    boolean timedOut = false; // Did the last poll() time out?

    for (;;) {
        int c = ctl.get();
        int rs = runStateOf(c);

        // Check if queue empty only if necessary.
        // 如果线程池正在关闭却队列已经为空，或者线程池已关闭，修改工作线程数量
        // 当前线程会自然运行结束
        if (rs >= SHUTDOWN && (rs >= STOP || workQueue.isEmpty())) {
            decrementWorkerCount(); // CAS机制
            return null;
        }

        int wc = workerCountOf(c);

        // Are workers subject to culling?
        boolean timed = allowCoreThreadTimeOut || wc > corePoolSize;

        // 此处用于维护核心线程数量
        if ((wc > maximumPoolSize || (timed && timedOut))
            && (wc > 1 || workQueue.isEmpty())) {
            if (compareAndDecrementWorkerCount(c))
                return null;
            continue;
        }

        try {
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

#### 任务的获取与工作线程间的“协议”
* 当线程池关闭时，获取任务方法将返回`null`，并关闭工作线程。
* 当工作线程执行完首个任务后，再次获取任务时将 *维护核心工作线程数量*

#### 工作线程退出流程
1. 统计线程池已完成任务数量（需要获取mainLock）
2. 在工作线程组中剔除当前线程
3. 尝试终止线程池
4. 

``` java
private void processWorkerExit(Worker w, boolean completedAbruptly) {
    // 如果当前工作线程异常，需要修正工作线程数量
    // 因为正常情形下（队列为空后），getTask方法发现队列为空时，已经修改了工作线程数量
    if (completedAbruptly) // If abrupt, then workerCount wasn't adjusted
        decrementWorkerCount();

    final ReentrantLock mainLock = this.mainLock;
    mainLock.lock();
    try {
        completedTaskCount += w.completedTasks;
        workers.remove(w);
    } finally {
        mainLock.unlock();
    }

    // 尝试终止线程池
    tryTerminate();

    int c = ctl.get();

    // 如果线程池没有被关闭，则需要维持最小的工作线程数:
    // 允许核心工作线超时，最小数量为0，否则最小数量为预测值corePoolSize
    // 特例：如果最小数量为0，但队列又不为空，那修正为1（因为总要有工作的线程吧）
    if (runStateLessThan(c, STOP)) {
        if (!completedAbruptly) { // 当前工作线程正常退出时，需要判断是否有足够多的核心工作线程
            int min = allowCoreThreadTimeOut ? 0 : corePoolSize;
            if (min == 0 && ! workQueue.isEmpty())
                min = 1;
            if (workerCountOf(c) >= min)
                return; // replacement not needed
        }

        // 线程池没有关闭 且 工作线程数量小于核心工作线程数，则需要再重新启动一个线程，保持核心工作线程数
        addWorker(null, false); // 线程补足
    }
}
```
如果线程池没有被关闭，则需要维持最小的工作线程数：允许核心工作线超时，最小数量为0，否则最小数量为预测值corePoolSize。
*特例*：如果此时最小数量为0，但队列不为空（有可能执行完队列中任务的同时，有并发线程提交了任务），那么修正为1（因为总要有工作的线程吧）


## 总结

* 时刻检查线程池状态：
* 状态的深刻要理解，因为线程池状态的检查无处不在，也需要时刻检查：
  * SHUTDOWN：不再接受新的任务（不再创建工作线程），但需要依次执行队列中任务*（特别是工作线程发生异常时，需要保证至少有1个工作线程去完成队列中任务，必须时还需重新创建工作线程）*
  * STOP：不再处理任务，立刻中断所有工作线程
* CAS（campare and set）机制与原子性：
  为了提高系统活跃度，整个框架总是先获取线程池状态：`c=ctl.get()`，然后循环`for(;;)`进行*compareAndSet*操作，确保*当前一系列操作的原子性*没有被破坏
* mainLock：工作线程workers、已完成任务数量completedTaskCount均是非线程安全的，通过`ReentrantLock`保证其线程安全


