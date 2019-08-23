---
layout: post
title: JAVA PROXY
subtitle: java 动态代理
date: 2019-08-22
categories: java
cover: 
tags: all 动态代理
---

java动态代理有JDK和CGLib两种方式实现

# JDK
JDK方式实现InvocationHandler接口，并且只能代理接口

```java
public interface People {
    public void sayHello();
}
```
```java
public class Chinese implements People{

    @Override
    public void sayHello() {
        System.out.println("你好世界");
    }
}
```
```java
import java.lang.reflect.InvocationHandler;
import java.lang.reflect.Method;
import java.lang.reflect.Proxy;

public class PeopleInvocationHandler implements InvocationHandler {

    private Object people;

    PeopleInvocationHandler(Object people){
        this.people = people;
    }
    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        System.out.println("before");
        Object res = method.invoke(people, args);
        System.out.println("after");
        return res;
    }

    public static void main(String[] args) {
        People zhangsan = new Chinese();
        PeopleInvocationHandler handler = new PeopleInvocationHandler(zhangsan);

        People proxy = (People) Proxy.newProxyInstance(zhangsan.getClass().getClassLoader(), zhangsan.getClass().getInterfaces(), handler);
        proxy.sayHello();
    }
}

/**
==============================
before
你好世界
after
*/
```
JDK代理会根据目标接口生成一个代理类，该代理类继承了proxy类，并且实现了被代理接口，这也就决定了JDK代理不能代理类

# CGLib
cglib 方式实现MethodInterceptor接口
```java
import org.springframework.cglib.proxy.Enhancer;
import org.springframework.cglib.proxy.MethodInterceptor;
import org.springframework.cglib.proxy.MethodProxy;

import java.lang.reflect.Method;

public class CGLibProxy implements MethodInterceptor {
    public <T> T getProxy(Class<T> cls){
        return (T) Enhancer.create(cls,this);
    }

    @Override
    public Object intercept(Object o, Method method, Object[] objects, MethodProxy methodProxy) throws Throwable {
        System.out.println("before");
        methodProxy.invokeSuper(o, objects);
        System.out.println("after");
        return null;
    }

    public static void main(String[] args) {
        CGLibProxy enhancer = new CGLibProxy();
        Chinese zhangsan = enhancer.getProxy(Chinese.class);
        zhangsan.sayHello();
    }
}
/**
==============================
before
你好世界
after
*/
```
CGLib采用的是用创建一个继承实现类的子类，用asm库动态修改子类的代码来实现的，所以可以用传入的类引用执行代理类，同时，也就决定了cglib不能代理final方法

# 区别
java动态代理是利用反射机制生成一个实现代理接口的匿名类，在调用具体方法前调用InvokeHandler来处理。核心是实现InvocationHandler接口，使用invoke()方法进行面向切面的处理，调用相应的通知。

而cglib动态代理是利用asm开源包，对代理对象类的class文件加载进来，通过修改其字节码生成子类来处理。核心是实现MethodInterceptor接口，使用intercept()方法进行面向切面的处理，调用相应的通知。

# 性能
调用少的情况下jdk代理更快，因为cglib需要生成class

调用多的情况下：因为jdk代理采用反射调用，所以在jdk6以前，cglib是更快的，jdk7、8对代理做了优化，到jdk8,jdk代理性能已经优于cglib了