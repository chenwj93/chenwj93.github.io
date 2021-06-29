---
layout: post
title: Java Synchronized Usage
subtitle: java synchronized 使用
date: 2021-06-28
categories: java
cover: 
tags: all java lock synchronized
---

## synchronized this
```
public class TestSychronizedThis implements Runnable {
    public static int signal = 1;

    public void methodA() throws InterruptedException {
        synchronized (this) {
            System.out.println("A start");
            Thread.sleep(5000);
            System.out.println("A end");
        }
    }

    public void methodB() {
        synchronized (this) {
            System.out.println("B start");
            System.out.println("B end");
        }
    }


    @Override
    public void run() {
        try {
            if (TestSychronizedThis.signal == 1) {
                this.methodA();
            } else {
                this.methodB();
            }
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }

    public static void main(String[] args) throws InterruptedException {
        TestSychronizedThis tst1 = new TestSychronizedThis();
        Thread t1 = new Thread(tst1);
        Thread t2 = new Thread(tst1);
        t1.start();
        Thread.sleep(500);
        TestSychronizedThis.signal = 2;
        t2.start();
    }
}

// out
A start
A end
B start
B end
```
结论：syncronized(this) 锁住的是对象，不同线程对同一对象的任一syncronized(this)访问将会阻塞

## synchronized function
```
public class TestSychronizedThis implements Runnable {
    public static int signal = 1;

    public synchronized void methodA() throws InterruptedException {
            System.out.println("A start");
            Thread.sleep(5000);
            System.out.println("A end");
    }

    public synchronized void methodB() {
            System.out.println("B start");
            System.out.println("B end");
    }


    @Override
    public void run() {
        try {
            if (TestSychronizedThis.signal == 1) {
                this.methodA();
            } else {
                this.methodB();
            }
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }

    public static void main(String[] args) throws InterruptedException {
        TestSychronizedThis tst1 = new TestSychronizedThis();
        Thread t1 = new Thread(tst1);
        Thread t2 = new Thread(tst1);
        t1.start();
        Thread.sleep(500);
        TestSychronizedThis.signal = 2;
        t2.start();
    }
}

// out
A start
A end
B start
B end
```
结论：syncronized function 锁住的是对象，不同线程对同一对象的任一syncronized function访问将会阻塞

同理，syncronized function 与syncronized(this)混用效果相同，原因是他们锁住的是同一对象
```
public class TestSychronizedThis implements Runnable {
    public static int signal = 1;

    public synchronized void methodA() throws InterruptedException {
            System.out.println("A start");
            Thread.sleep(5000);
            System.out.println("A end");
    }

    public void methodB() {
        synchronized (this) {
            System.out.println("B start");
            System.out.println("B end");
        }
    }


    @Override
    public void run() {
        try {
            if (TestSychronizedThis.signal == 1) {
                this.methodA();
            } else {
                this.methodB();
            }
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }

    public static void main(String[] args) throws InterruptedException {
        TestSychronizedThis tst1 = new TestSychronizedThis();
        Thread t1 = new Thread(tst1);
        Thread t2 = new Thread(tst1);
        t1.start();
        Thread.sleep(500);
        TestSychronizedThis.signal = 2;
        t2.start();
    }
}

// out
A start
A end
B start
B end
```

## synchronized(Object)
```
public class TestSychronizedThis implements Runnable {
    public static int signal = 1;
    public Integer lock = 0;

    public TestSychronizedThis(Integer lock) {
        this.lock = lock;
    }

    public synchronized void methodA() throws InterruptedException {
        synchronized(lock) {
            System.out.println("A start");
            Thread.sleep(5000);
            System.out.println("A end");
        }
    }

    public void methodB() {
        synchronized (lock) {
            System.out.println("B start");
            System.out.println("B end");
        }
    }


    @Override
    public void run() {
        try {
            if (TestSychronizedThis.signal == 1) {
                this.methodA();
            } else {
                this.methodB();
            }
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }

    public static void main(String[] args) throws InterruptedException {
        Integer lock = 1;
        TestSychronizedThis tst1 = new TestSychronizedThis(lock);
        // 新建对象，但共享lock对象
        TestSychronizedThis tst2 = new TestSychronizedThis(lock);
        Thread t1 = new Thread(tst1);
        Thread t2 = new Thread(tst2);
        t1.start();
        Thread.sleep(500);
        TestSychronizedThis.signal = 2;
        t2.start();
    }
}

// out
A start
A end
B start
B end
```
结论：synchronized(Object)锁住的是Object，任意线程对同一synchronized(Object)块的访问都是同步的

## synchronized static
```
public class TestSychronizedThis implements Runnable {
    public static int signal = 1;

    public synchronized static void methodA() throws InterruptedException {
            System.out.println("A start");
            Thread.sleep(5000);
            System.out.println("A end");
    }

    public synchronized static void methodB() {
            System.out.println("B start");
            System.out.println("B end");
    }


    @Override
    public void run() {
        try {
            if (TestSychronizedThis.signal == 1) {
                this.methodA();
            } else {
                this.methodB();
            }
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }

    public static void main(String[] args) throws InterruptedException {
        Integer lock = 1;
        TestSychronizedThis tst1 = new TestSychronizedThis();
        TestSychronizedThis tst2 = new TestSychronizedThis();
        Thread t1 = new Thread(tst1);
        Thread t2 = new Thread(tst2);
        t1.start();
        Thread.sleep(500);
        TestSychronizedThis.signal = 2;
        t2.start();
    }
}

// out
A start
A end
B start
B end
```
结论：毫无疑问，静态方法被锁则类被锁住，不同实例的类锁代码块访问是同步的

不过需要注意的是类对象与实例对象的区别，类锁和对象锁是互不影响的，不会相互阻塞，以下代码为例：
```
public class TestSychronizedThis implements Runnable {
    public static int signal = 1;

    public synchronized static void methodA() throws InterruptedException {
            System.out.println("A start");
            Thread.sleep(5000);
            System.out.println("A end");
    }

    public synchronized  void methodB() {
            System.out.println("B start");
            System.out.println("B end");
    }


    @Override
    public void run() {
        try {
            if (TestSychronizedThis.signal == 1) {
                this.methodA();
            } else {
                this.methodB();
            }
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }

    public static void main(String[] args) throws InterruptedException {
        Integer lock = 1;
        TestSychronizedThis tst1 = new TestSychronizedThis();
        TestSychronizedThis tst2 = new TestSychronizedThis();
        Thread t1 = new Thread(tst1);
        Thread t2 = new Thread(tst2);
        t1.start();
        Thread.sleep(500);
        TestSychronizedThis.signal = 2;
        t2.start();
    }
}

//out
A start
B start
B end
A end
```

## synchronized(.class)
```
public class TestSychronizedThis implements Runnable {
    public static int signal = 1;

    public  void methodA() throws InterruptedException {
        synchronized(TestSychronizedThis.class) {
            System.out.println("A start");
            Thread.sleep(5000);
            System.out.println("A end");
        }
    }

    public void methodB() {
        synchronized(TestSychronizedThis.class) {
            System.out.println("B start");
            System.out.println("B end");
        }
    }


    @Override
    public void run() {
        try {
            if (TestSychronizedThis.signal == 1) {
                this.methodA();
            } else {
                this.methodB();
            }
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }

    public static void main(String[] args) throws InterruptedException {
        Integer lock = 1;
        TestSychronizedThis tst1 = new TestSychronizedThis();
        TestSychronizedThis tst2 = new TestSychronizedThis();
        Thread t1 = new Thread(tst1);
        Thread t2 = new Thread(tst2);
        t1.start();
        Thread.sleep(500);
        TestSychronizedThis.signal = 2;
        t2.start();
    }
}

// out
A start
A end
B start
B end
```
结论：不同实例的synchronized(.class)代码块访问是同步的

## 总结

- synchronized是锁住一个对象，所有线程对同一对象的访问是同步的
- 对象可以是类对象和实例对象
