---
title: Java中的多线程（二）：同步机制
date: 2016-03-15 11:32:44
tags:
- Concurrence
- Thread
- Synchronization
categories: Java-Core
toc: true
---

进程中的多个线程间往往需要通信，共同完成或维护进程的目的。线程间的通信主要通过共享数据（基础类型数据，对象引用等)，这种共享数据的方式会导致两种潜在发生的错误：线程干扰（thread interference）和内存一致性错误（memory consistency errors）。而同步（synchronization）机制应运而生，用于解决上面可能出现的错误。

同步机制虽然可以解决线程干扰和内存一致性问题，但也可能带来其他问题：线程竞争（thread contention）。当多个线程尝试访问同一块资源时产出了竞争关系，有可能会导致线程被挂起或者死锁。

<!-- more -->

## 线程干扰
线程干扰是指，别的线程会影响当前线程执行结果的正确性。
举个常见的栗子：`i++`。这个自增操作对应的jvm指令大概是这样：

1. 内存中获取 i 的值
2. 对 i 执行 i+1 操作
3. 将 i+1 写入内存

如果两个线程同时对某一成员执行自增操作，考虑以下场景：两个线程同时从内存中取得相同的值，执行递增操作后，写入内存的时间发生了差异。那么后执行的线程必然覆盖提前结束线程的操作结果。

``` java
class Counter {
    private int count = 0;

    public  void increase() {
        for (int i=0; i<100000; i++) {
            count++;
        }
    }
    
    public  void reduce() {
        for (int i=0; i<100000; i++) {
            count--;
        }
    }
    
    public int getCount() {
        return count;
    }
}

@Test
public void testInterference() throws Exception {
    Counter counter = new Counter();
    
    new Thread(() -> counter.increase());
    
    counter.reduce();
    
    Thread.sleep(1000);
    // 断言将会失败
    assertEquals(0, counter.getCount());
}
```

## 内存一致性问题
每个线程有自己的栈，为了提供访问效率，一般会将进程中堆上的数据做一份缓存放在自己的栈上面。那么有可能在短暂时间内会导致一个线程的修改结果对另一个线程不可见。Java提供了[happens-before](https://docs.oracle.com/javase/specs/jls/se7/html/jls-17.html#jls-17.4.5)机制来避免内存一致性问题。

### happens-before
只有当一个写操作与读操作存在happens-before关系时，才能保证此写操作的结果对那个读操作可见。同步（synchronized）、volatile关键字、Thread.start()，Thread.join()都会建立happens-before关系。

* 一个线程中，每个操作与当前线程中后面的操作存在happens-before关系。
* 未上锁的同步区块或方法与后面的锁定区块或方法存在happens-before关系（注意：这种关系是可以传递的）
* 一个*volatile*域的写操作与随后的读操作存在happens-before关系。
* 一个主线程与它启动的其他线程存在happens-before关系。
* 一个线程与成功join的其他线程存在happens-before关系。

## 内部锁与监视器
同步机制的实现是通过监视器来实现：*Java中的每个对象都有一个可以被锁定或者解锁的监视器*。任何时间段里有且只可能有一个线程拥有这个监视器的锁。任何尝试对已经锁定的监视器进行再次锁定的线程都会被阻塞（当然不包括已经含有此锁的线程），直到他们获得监视器的锁。

*同步语句*会计算对象的引用，并尝试对对象的监视器进行锁定操作，直到锁定操作成功后才会执行后面的操作。当同步内容执行完毕后，又会自动对监视器进行解锁操作。

当一个*同步方法*被调用时，会自动进行锁定监视器的操作，同样只有锁定操作成功后才会执行方法体内的操作。对于*实例方法*，锁定的是被调用实例的监视器。而对于*静态方法*，锁定的是被调用的`Class`对象。

Java中并不会避免或者检测*死锁条件*，所以应用程序必要时需要自己避免死锁。

## 同步方法

方法同步的影响：

* 不允许对一个对象的两个方法交替调用。也就是说当一个线程正在执行某对象的同步方法时，其他调用改方法的线程将被阻塞或挂起，直到那个正在执行的线程执行完毕。
* 当退出同步方法时，将与随后对此方法的调用自动建立happens-before关系。保证对象状态的改变对随后的线程可见。

构造方法不能用于同步，因为没有意义，为什么呢？（因为只有当前线程才能创建它）。

## 同步语句
同步方法是对整个被调用的（this）*实例*或者*Class*的监视器进行锁定，而同步语句不一定要对当前被调用的实例锁定，它可以锁定指定的对象，这将非常有帮助。比如一个类中有两个成员，foo和bar，但它们并不会同时使用。所有对它们的更新都需要同步，但并不意味着两个线程不可以交叉分别更新它们。我们可以单独对它们的监视器进行锁定，针对基本类型可以分别创建一个对象锁。
``` java
class MyLock {
    private int foo;
    private Object lockForFoo = new Object();
    
    private Long bar = 0L;
    
    public void increaseFoo() {
        synchronized (lockForFoo) {
            foo++;
        }
    }
    
    public void increaseBar() {
        synchronized (bar) {
            bar++;
        }
    }
    
    public void print() {
        System.out.format("foo=%d, bar=%d%n", foo, bar);
    }
}

@Test
public void test() throws InterruptedException {
    MyLock lock = new MyLock();
    
    new Thread(() -> lock.increaseFoo()).start();;
    new Thread(() -> lock.increaseBar()).start();;
    
    Thread.sleep(10);
    lock.print();
}
```

## 多次同步
大家已经知道一个线程只能等待另一个线程解锁监视器后才能调用其同步方法，那如果一个同步方法中会调用另外一个同步方法会怎样？答案是可以的，该线程已经获得了这个实例的内部锁，可以再次进入别的同步方法。

## 原子访问
如上所示，`c++`在多线程环境容易导致并发问题，其中一个原因为这并不是一个原子性的操作，它被分解成3个指令来执行。对于这种基本类型的增减操作，Java提供了相关的`Atomic`原子类。

原子操作，意味着要么被全部执行，要么什么都不执行。虽然原子操作可以避免线程干扰的问题，但是不是在多线程环境下就一定不会有线程安全问题呢？答案是否定的，因为可能导致内存不一致问题。这时`volatile`就派上用场了。所以在使用Atomic类是需要注意内存一致性问题。










