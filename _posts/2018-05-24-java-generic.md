---
layout: post
title: Java Generic
subtitle: java 泛型
date: 2019-07-16
categories: java
cover: 
tags: all 泛型
---

> 库存 java-泛型

### 泛型类
```java
public class genericity {
    class Generic<T extends Number>{
        //key这个成员变量的类型为T,T的类型由外部指定
        private T key;

        public Generic(T key) { //泛型构造方法形参key的类型也为T，T的类型由外部指定
            this.key = key;
        }

        public T getKey(){ //泛型方法getKey的返回值类型为T，T的类型由外部指定
            return key;
        }
    }

    public void test1() {
        Generic<Integer> i = new Generic<Integer>(1);
        Generic<? extends Number> i2 = new Generic<Float>(2f);
    }
}
```

### 泛型接口
```java
public class genericity {
    /**
     * 未传入泛型实参时，与泛型类的定义相同，在声明类的时候，需将泛型的声明也一起加到类中
     * 即：class FruitGenerator<T> implements Generator<T>{
     * 如果不声明泛型，如：class FruitGenerator implements Generator<T>，编译器会报错："Unknown class"
     */
    class FruitGenerator<T> implements Generator<T>{
        @Override
        public T next() {
            return null;
        }
    }

    /**
     * 传入泛型实参时：
     * 定义一个生产器实现这个接口,虽然我们只创建了一个泛型接口Generator<T>
     * 但是我们可以为T传入无数个实参，形成无数种类型的Generator接口。
     * 在实现类实现泛型接口时，如已将泛型类型传入实参类型，则所有使用泛型的地方都要替换成传入的实参类型
     * 即：Generator<T>，public T next();中的的T都要替换成传入的String类型。
     */
    class MeatGenerator implements Generator<String> {

        private String[] fruits = new String[]{"Apple", "Banana", "Pear"};

        @Override
        public String next() {
            Random rand = new Random();
            return fruits[rand.nextInt(3)];
        }
    }

    public void test2(){
        Generator<Integer> f = new FruitGenerator<Integer>();
        Generator<String> m = new MeatGenerator();
        Generator<? extends String> m2 = new MeatGenerator();
    }
}
```
### 泛型方法
```java
public <T> T genericMethod(Class<T> tClass) throws InstantiationException,
            IllegalAccessException {
        T instance = tClass.newInstance();
        return instance;
    }

    public void test6() throws ClassNotFoundException, IllegalAccessException, InstantiationException {
        Object obj = genericMethod(Class.forName("com.cwj.learn_1.genericity"));
    }
```
详细说明：
```java
public class GenericTest {
   //这个类是个泛型类，在上面已经介绍过
   public class Generic<T>{     
        private T key;

        public Generic(T key) {
            this.key = key;
        }

        //我想说的其实是这个，虽然在方法中使用了泛型，但是这并不是一个泛型方法。
        //这只是类中一个普通的成员方法，只不过他的返回值是在声明泛型类已经声明过的泛型。
        //所以在这个方法中才可以继续使用 T 这个泛型。
        public T getKey(){
            return key;
        }
        
        //在泛型类中声明了一个泛型方法，使用泛型E，这种泛型E可以为任意类型。可以类型与T相同，也可以不同。
        //由于泛型方法在声明的时候会声明泛型<E>，因此即使在泛型类中并未声明泛型，编译器也能够正确识别泛型方法中识别的泛型。
        public <E> void show_3(E t){
            System.out.println(t.toString());
        }

        /**
         * 这个方法显然是有问题的，在编译器会给我们提示这样的错误信息"cannot reslove symbol E"
         * 因为在类的声明中并未声明泛型E，所以在使用E做形参和返回值类型时，编译器会无法识别。
        public E setKey(E key){
             this.key = keu
        }
        */
    }

    /** 
     * 这才是一个真正的泛型方法。
     * 首先在public与返回值之间的<T>必不可少，这表明这是一个泛型方法，并且声明了一个泛型T
     * 这个T可以出现在这个泛型方法的任意位置.
     * 泛型的数量也可以为任意多个 
     *    如：public <T,K> K showKeyName(Generic<T> container){
     *        ...
     *        }
     */
    public <T> T showKeyName(Generic<T> container){
        System.out.println("container key :" + container.getKey());
        //当然这个例子举的不太合适，只是为了说明泛型方法的特性。
        T test = container.getKey();
        return test;
    }
    
    

    //这也不是一个泛型方法，这就是一个普通的方法，只是使用了Generic<Number>这个泛型类做形参而已。
    public void showKeyValue1(Generic<Number> obj){
        Log.d("泛型测试","key value is " + obj.getKey());
    }

    //这也不是一个泛型方法，这也是一个普通的方法，只不过使用了泛型通配符?
    //同时这也印证了泛型通配符章节所描述的，?是一种类型实参，可以看做为Number等所有类的父类
    public void showKeyValue2(Generic<?> obj){
        Log.d("泛型测试","key value is " + obj.getKey());
    }

     /**
     * 这个方法是有问题的，编译器会为我们提示错误信息："UnKnown class 'E' "
     * 虽然我们声明了<T>,也表明了这是一个可以处理泛型的类型的泛型方法。
     * 但是只声明了泛型类型T，并未声明泛型类型E，因此编译器并不知道该如何处理E这个类型。
    public <T> T showKeyName(Generic<E> container){
        ...
    }  
    */

    /**
     * 这个方法也是有问题的，编译器会为我们提示错误信息："UnKnown class 'T' "
     * 对于编译器来说T这个类型并未项目中声明过，因此编译也不知道该如何编译这个类。
     * 所以这也不是一个正确的泛型方法声明。
    public void showkey(T genericObj){

    }
    */

    public static void main(String[] args) {


    }
}
```




### 泛型通配符
以下这种写法中最大的问题是showKeyValue的参数太固定了，只有Generic<Number>这一种参数可以通过编译
```java
public class genericity {
    public void showKeyValue(Generic<Number> obj){
        System.out.println(("泛型测试"+" key value is " + obj.getKey()));
    }

    public void test3(){
        Generic<Integer> gInteger = new Generic<Integer>(123);
        Generic<Number> gNumber = new Generic<Number>(456);

        showKeyValue(gNumber);

        // showKeyValue这个方法编译器会为我们报错：Generic<java.lang.Integer>
        // cannot be applied to Generic<java.lang.Number>
        //showKeyValue(gInteger);
    }
}
```
使用通配符解决问题：
```java
public void showKeyValue1(Generic<?> obj){
    Log.d("泛型测试","key value is " + obj.getKey()));
}
```
#### 通配符上下界
```java
    class A{

    }

    class B extends A{

    }

    class C extends B{

    }

    class temp<T> {
        private T item;

        public temp(T t) {
            item = t;
        }

        public void set(T t) {
            item = t;
        }

        public T get() {
            return item;
        }
    }

    public void test5(){
        temp<? extends B> b = new temp<B>(new C());
        //以下写法报错
        //b.set(new B());

        temp<? super B> b1 = new temp<B>(new B());
        b1.set(new B());
        b1.set(new C());

        //get只能用Object接收
        //B b2 = b1.get();
        Object b2 = b1.get();

        //以下写法与上界extends的副作用一样
        temp<?> b3 = new temp<Object>(new B());
        // 不允许
        b3.set(new B());
    }
```
- 上界（extends）代表从子类往上溯的最高父类
- 下界（super）代表从父类往子类下溯的最低子类

当设置上界时，编译器并不知道T究竟是什么类型，比如上例，设置B类为上界，但是T可能是B，也可能是C，这个时候如果放一个B进去，编译器是不允许通过的。因此，上界对象不允许set

当设置下界时，编译器虽然不知道T究竟是什么类型，但是可以肯定一定是下界以下的子类的父类，上例中 B为下界时，T可能是A， 也可能是B，但是C一定是它们的子类，因此可以使用set函数，但是取出来时就只能使用Object类接收了。

**因此，此处遵循PECS(Producer Extends Consumer Super)原则：**
- 频繁往外读取内容的，适合用上界Extends
- 经常往里插入的，适合用下界Super