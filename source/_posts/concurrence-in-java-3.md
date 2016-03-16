---
title: Java中的多线程（三）：活跃度
date: 2016-03-16 01:58:19
tags: 
- Concurrence
- Thread
- Liveness
categories: Java-Core
toc: true
---

一个并发程序的及时执行能力叫做活跃度（liveness）。活跃度问题一般包括死锁（deadlock）、饥饿（starvation）和活锁（liveness）。

<!-- more -->

## 死锁（deadlock)
死锁，指两个或更多的线程被永久阻塞，等待彼此进行解锁。

## 饥饿（starvation）
饥饿，指某个线程长时间内无法获得资源而处于阻塞状态，这种现象常常是由于其他“贪婪”线程长时间占用资源导致。

## 活锁（livelock）
活锁，指一个线程的操作或响应其他线程，而其他线程又会响应另外线程，这时候可能导致活锁。活锁同样会导致程序无法进行，但跟死锁不同的是，它并没有阻塞

## 守护区块（guarded）
线程间经常需要协调他们的活动。最常用的协调习惯就是守护区块：该区块轮询一个条件，直到满足后才会执行。

最常见的错误用法为：
``` java
class Guarded {

    private boolean flag = false;
    
    public void run() {
        // 等待
        while(!flag){
            System.out.println("waiting...");
        }
        
        System.out.println("Run done!");
    }
    
    public void activate() {
        flag = true;
    }
}

public void testGuarded() {
    Guarded guarded = new Guarded();
    
    new Thread(() -> guarded.run()).start();
    new Thread(() -> guarded.activate()).start();
}
```
上面的程序也会正常运行，但是开销是巨大的，应该那个轮询会一直进行打印“waiting...”。正确的做法是通过“等待-通知”模式：
``` java
class Guarded {

    private boolean flag = false;
    
    public synchronized void run() throws InterruptedException {
        while(!flag){
            System.out.println("waiting...");

            // 等待
            wait();
        }
        
        System.out.println("Run done!");
    }
    
    public synchronized void activate() throws InterruptedException {
        flag = true;
        
        // 通知
        notify();
        System.out.println("notify...");
    }
}
```
`wait()`常常与*synchronized方法*一起使用，用来获取内部锁。当*wait*方法被调用时，这个线程会释放内部锁并挂起。当然其他线程同样可以调用此同步方法，获得内部锁并再次执行*wait*然后将自己挂起。当将来有个线程执行`nitifyAll()`时将会通知之前*所有*被挂起的线程。

*注意*：nitifyAll并不会*同时*唤醒所有等待中的线程，因为毕竟内部锁（监视锁）只有一把，有且只有一个线程获得，然后执行剩下的操作。应该是其他线程依次被唤醒，但没有固定的顺序，依赖CPU的算法。

## 生产者与消费者问题
``` java
class Drop {
    private static final String DONE = "Done";
    
    private String message;
    // 注意这里的初始化
    private boolean empty = true;
    
    public synchronized String take() {
        while (empty) {
            try {
                wait();
            } catch (InterruptedException e) {
            }
        }
        
        empty = true;
        notifyAll();
        return message;
    }
    
    public synchronized void put(String message) {
        while (!empty) {
            try {
                wait();
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
        this.message = message;
        empty = false;
        notify();
    }
}

class Producer implements Runnable {
    String dataForProduct = "simply retrieves the messages and prints them out";
    Random random = new Random();
    
    Drop drop;
    Producer(Drop drop) {
        this.drop = drop;
    }
    
    @Override
    public void run() {
        for (String message : dataForProduct.split(" ")) {
            drop.put(message);
            System.out.println("Product data: " + message);
            
            try {
                Thread.sleep(random.nextInt(500));
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
        
        drop.put(Drop.DONE);
    }
}

class Consumer implements Runnable {
    Drop drop;
    Consumer(Drop drop) {
        this.drop = drop;
    }
    
    @Override
    public void run() {
        for (String message = drop.take(); !message.equals(Drop.DONE); message = drop.take()) {
            System.out.println("Received data : " + message);
        }
    }
}

@Test
public void test() throws InterruptedException {
    Drop drop = new Drop();
    
    new Thread(new Producer(drop)).start();
    new Thread(new Consumer(drop)).start();
    
    Thread.sleep(3000);
}
```



