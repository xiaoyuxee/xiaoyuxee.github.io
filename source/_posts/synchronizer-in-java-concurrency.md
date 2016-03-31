---
title: Java中的多线程（四）：同步器
date: 2016-03-31 00:15:18
tags:
- Concurrence
- Thread
- Synchronizer
categories: Java-Core
toc: true
---

多线程编程中经常会遇到线程协调问题。比如经典的生产者-消费者模式，生产者和消费者的工作需要通过一个作业队列来协调：当队列中有作业时消费者才会从队列中取出一个作业进行消费，否则将一直处于等待状态。

最基础的线程协调可以通过同步机制与`wait()`结合来实现：在某个对象中设置一个标记，当修改标记后通过`notifyAll()`来通知其他等待的线程。

幸运的是Java提供一些同步器来协调线程间的控制流。同步器内部封装了一些状态，这些状态将决定执行同步器的线程是继续执行还是等待。常用的同步器有阻塞队列（BlockingQueue）、闭锁（Latch）、信号量（Semaphore）和栅栏（Barrier）。

<!-- more -->

## 闭锁（CountDownLatch）
[CountDownLatch](http://docs.oracle.com/javase/8/docs/api/java/util/concurrent/CountDownLatch.html)用于实现：一个或多个线程等待，直到其他线程完成某些操作。

CountDownLatch通过一个`count`来初始化，而`await()`方法将一直等待直到`count`的值变为0。CountDownLatch的状态无法重置。当需要重置功能时可以考虑使用栅栏[CyclicBarrier](http://docs.oracle.com/javase/8/docs/api/java/util/concurrent/CyclicBarrier.html)。

CountDownLatch常常用作开关或者一扇门，使*所有线程在门前等待，直到门被开启*。可以通过闭锁来启动一组操作，或者等待一组操作的结束。郑州

``` java
public class LatchTest {

    class Soldier implements Runnable {
        private final CountDownLatch gate;
        
        public Soldier(CountDownLatch gate) {
            this.gate = gate;
        }
        
        @Override
        public void run() {
            try {
                System.out.println("Waiting for gate open...");
                gate.await();
            } catch (InterruptedException e) {
                return;
            }
            
            fight();
        }
        
        private void fight() {
            System.out.println("Fight... ");
        }
    }
    
    @Test
    public void test() throws InterruptedException {
        
        CountDownLatch gate = new CountDownLatch(1);
        int soldierCount = 5;
        
        Executor executor = Executors.newCachedThreadPool();
        for (int i=0; i<soldierCount; i++) {
            executor.execute(new Soldier(gate));
        }
        
        Thread.sleep(500);
        
        System.out.println("gate opened!");
        gate.countDown();
        
        Thread.sleep(500);
    }

}
```

## 栅栏（Barrier）
[Barrier](http://docs.oracle.com/javase/8/docs/api/java/util/concurrent/CyclicBarrier.html)跟CountDownLatch类似，都能阻塞一组线程直到某个事件发生。关键区别在于：*CountDownLatch等待的是事件，count通过`countDown()`变为0的事件。而Barrier等待的是其他线程。*

Barrier常常用在并行迭代算法中：将一个问题拆分成一系列相互独立的子问题。如果`await()`调用超时或者被阻塞的线程被中断，那么栅栏被认为是打破了，所有其他阻塞的线程终止并抛出`BrokenBarrierException`。

CyclicBarrier的构造函数支持传递一个Runable，当成功通过栅栏时会在一个子任务线程中执行它。


## 信号量（Semaphore）
计数信号量用来控制*同时*访问的某个资源的操作数量或实现某种*资源池*，如数据库连接池。

初始值为1的信号量可以用作互斥锁，它与内置锁类似，但不可重入。
