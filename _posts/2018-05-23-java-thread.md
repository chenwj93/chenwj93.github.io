---
layout: post
title: Java Thread
subtitle: java 多线程
date: 2019-07-29
categories: java
cover: 
tags: all java 多线程
---

> 库存 java-多线程

- Thread 是类，并且实现了Runnable接口
- Runnable接口没有start()方法
- 之所以需要继承Thread类或是实现Runnable接口，是因为new Thread()方法接受的是Runnable实例（自己理解）


#### 继承Thread
```java
public class testThread extends Thread {

    private int tickets = 10;

    testThread(String name){
        super(name);
    }
    
    
    testThread(){
    }

    @Override
    public void run() {

        for (int i = 0; i <= 100; i++) {
            if(tickets>0){
                System.out.println(Thread.currentThread().getName()+"--卖出票：" + tickets--);
            }
        }
    }

    //线程独立，不共享资源
    public static void main(String[] args) {
        Thread thread1 = new testThread("窗口1");
        Thread thread2 = new testThread("窗口2");
        Thread thread3 = new testThread("窗口3");

        thread1.start();
        thread2.start();
        thread3.start();
        //每个线程都独立，不共享资源，每个线程都卖出了10张票，总共卖出了30张。如果真卖票，就有问题了。
    }
    
   //线程共享资源
    public static void main(String[] args) {
        Thread thread1 = new testThread();

        new Thread(thread1,"窗口1").start();
        new Thread(thread1,"窗口2").start();
        new Thread(thread1,"窗口3").start();
    }

}
```

#### 实现Runnable
```java
public class testRunnable implements Runnable {

    private int tickets = 10;

    @Override
    public void run() {

        for (int i = 0; i <= 100; i++) {
            if(tickets>0){
                System.out.println(Thread.currentThread().getName()+"--卖出票：" + tickets--);
            }
        }
    }

    // 不共享资源
    public static void main(String[] args) {
        testRunnable myRunnable1 = new testRunnable();
        Thread thread1 = new Thread(myRunnable1, "窗口一");
        testRunnable myRunnable2 = new testRunnable();
        Thread thread2 = new Thread(myRunnable2, "窗口二");
        testRunnable myRunnable3 = new testRunnable();
        Thread thread3 = new Thread(myRunnable3, "窗口三");

        thread1.start();
        thread2.start();
        thread3.start();
    }

    // 共享资源
    public static void main(String[] args) {
        testRunnable myRunnable = new testRunnable();
        Thread thread1 = new Thread(myRunnable, "窗口一");
        Thread thread2 = new Thread(myRunnable, "窗口二");
        Thread thread3 = new Thread(myRunnable, "窗口三");

        thread1.start();
        thread2.start();
        thread3.start();
    }

}
```

**Thread与Runnable的优劣总的来说只有一点**
- **实现接口更灵活，毕竟java不支持多继承**
- 至于资源共享问题，并非thread或Runnable本身造成的，只是因为写法的差异，关键点在于**实例化了几个对象**而已。