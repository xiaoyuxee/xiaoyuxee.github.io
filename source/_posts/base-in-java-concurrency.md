---
title: Java中的多线程（一）：线程的基本概念和使用
date: 2016-03-15 00:03:52
tags:
- Concurrency
- Thread
categories: Java-Core
toc: true
---
日常生活中常常会遇到并发场景，比如你浏览网页的同时，可能同时会受到QQ好友的消息。接下来的一些列文章将介绍如何在Java语言中进行多线程的编程。

本文介绍线程的一些基本概念和操作，如什么是线程，如何创建以及线程的睡眠与中断、join。

<!-- more -->

## 什么是进程，什么是线程
线程有时候被叫做轻量级的进程，创建一个线程的代销远小于一个进程。线程是进程中最小的运行单元，一个进程中至少包含一个线程。线程间可以共享资源。

而进程通常包含一个独立的运行环境，每个进程有自己的内存空间。

## 线程（Thread）
Java中每一个线程都是一个[Thread](https://docs.oracle.com/javase/8/docs/api/java/lang/Thread.html)的实例。

### 线程的创建
#### 实现Runable
``` java
public class ThreadTest {

    class HelloRunable implements Runnable {

        @Override
        public void run() {
            System.out.println("Hello Multithread!");
        }
    }
    
    @Test
    public void testCreateThread() {
        new Thread(new HelloRunable()).start();
    }

}
```
or 使用Lambda表达式
``` java
new Thread(() -> System.out.println("Hello Multithread!")).start();
```

#### 继承Thread
`Thread`实现了`Runable`接口，所以可以覆盖它的run()方法来创建一个线程。run方法的实现为：
``` java
public class Thread implements Runnable {
    private Runnable target;

    public void run() {
        if (target != null) {
            target.run();
        }
    }
}
```
``` java
public void testCreateThread() {
    new Thread() {

        @Override
        public void run() {
            System.out.println("Hello Multithread!");
        }
    }.start();
}
```

#### start与run
start方法有jvm实现，创建一个线程并调用run方法。而run方法只不过是Thread的一个普通方法，单独调用时也会普通的调用，仍然在主线程线程中。

#### 如何选择创建一个线程
建议通过实现`Runable`接口：因为Java语言是不支持多继承的，如果是通过继承`Thread`来实现多线程，那么灵活性也将受到限制。

### 线程睡眠
``` java
public void testThreadSleep() throws InterruptedException {
    Thread.sleep(1000);
    
    System.out.println("Bingo...");
}
```

### 线程中断
Thread中的很多方法都可能抛出`InterruptedException`异常，当然也可以通过`interrupt`方法来中断某个线程，那么*支持线程中断*就很重要了。

#### 支持线程中断
两种方式：捕捉异常 或 查询中断状态

#### 线程的中断状态
线程的中断机制是通过内部的标记来实现，也称中断状态。

中断线程可以通过线程的`interrupt`方法来中断当前线程。

查询线程的中断状态同样可以`Thread.interrupt`静态方法，也可以通过Thread类的`interrupted`方法来查询。不过这里有个需要注意的地方：
*前者会清空中断状态，而后者不会*。

### joins
Thead的`join`方法会让一个线程等待另一个线程完成。当在一个线程中调用另一个线程的`join`方法后，当前线程会停止执行后面的语句，直到另外一个线程完成。

``` java
public void testInterrupt() {
    Thread interrupt = new Thread(new Runnable() {
        
        @Override
        public void run() {
            try {
                Thread.sleep(1000);
            } catch (InterruptedException e) {
                return;
            }
            System.out.println("sub over");
        }
    });

    interrupt.start();
    // interrupt.join();

    System.out.println("main over");
 }
```

运行上面这个demo，你在控制台将看不到打印"sub over"：因为主线程没有睡眠，会比子线程先运行结束。除非将注释的代码放开。

