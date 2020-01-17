---
title: 从volatile第一个特性看JVM内存模型(JMM)
date: 2019-10-16 17:46:45
tags: 
- JVM
- JMM
categories: JMM
---

## 写在前面

### volatile两大特性

- 保证变量的可见性：当一个被volatile关键字修饰的变量被一个线程修改的时候，其他线程可以立刻得到修改之后的结果。当一个线程向被volatile关键字修饰的变量写入数据的时候，虚拟机会强制它被值刷新到主内存中。当一个线程用到被volatile关键字修饰的值的时候，虚拟机会强制要求它从主内存中读取。
-  屏蔽指令重排序：指令重排序是编译器和处理器为了高效对程序进行优化的手段，它只能保证程序执行的结果时正确的，但是无法保证程序的操作顺序与代码顺序一致。这在单线程中不会构成问题，但是在多线程中就会出现问题。非常经典的例子是在单例方法中同时对字段加入volatile，就是为了防止指令重排序。

这篇文章主要从其第一个特性浅析JVM的内存模型。为了了解JVM内存模型，我们首先要清楚[JVM的内存结构](https://styyt.github.io/2019/09/14/%E5%8D%87%E7%BA%A71.8%E5%AF%B9gc%E7%9A%84%E5%BD%B1%E5%93%8D/#jvm内存结构变化)。

## 示例代码

以下代码可以简单看出volatile保证了变量可见性

```java
public class VolatileOne extends Thread{
    //测试变量
	//如果没有加volatile修饰，则线程内的循环会变成死循环
	//加了volatile后，main线程更新了变量值，会立即使其他线程工作内存中的变量缓存失效
    //使得当前线程从主内存取值，从而跳出循环
    //public volatile  Boolean flag = true;
	public Boolean flag = true;
    //无限循环,等待flag变为true时才跳出循环
    public void run() {
        while (flag){
        };
        System.out.println("停止了");
    }

    public static void main(String[] args) throws Exception {
    	VolatileOne v1= new VolatileOne();v1.start();
        System.out.println("当前flag是"+v1.flag);
        //sleep以让v1开始执行循环
        Thread.sleep(100);
        v1.flag = false;
        System.out.println("当前flag是"+v1.flag);
    }
}
```

## 浅析Java内存模型（JMM）

- 补充

  看我之前的文章中曾经描述过：在JVM内部，Java内存模型把内存分成了两部分：线程栈区和堆区，JVM中运行的每个线程都拥有自己的线程栈，线程栈包含了当前线程执行的方法调用相关信息，我们也把它称作调用栈。随着代码的不断执行，调用栈会不断变化。

  详细点来说

  所有原始类型(boolean,byte,short,char,int,long,float,double)的局部变量都直接保存在线程栈当中，对于它们的值各个线程之间都是独立的。对于原始类型的局部变量，一个线程可以传递一个副本给另一个线程，当它们之间是无法共享的。
  堆区包含了Java应用创建的所有对象信息，不管对象是哪个线程创建的，其中的对象包括原始类型的封装类（如Byte、Integer、Long等等）。不管对象是属于一个成员变量还是方法中的局部变量，它都会被存储在堆区。
  一个局部变量如果是原始类型，那么它会被完全存储到栈区。 一个局部变量也有可能是一个对象的引用，这种情况下，这个本地引用会被存储到栈中，但是对象本身仍然存储在堆区。
  对于一个对象的成员方法，这些方法中包含局部变量，仍需要存储在栈区，即使它们所属的对象在堆区。 对于一个对象的成员变量，不管它是原始类型还是包装类型，都会被存储到堆区。Static类型的变量以及类本身相关信息都会随着类本身存储在堆区。

- Java内存模型（JMM）

  JMM定义了Java 虚拟机(JVM)在计算机内存(RAM)中的工作方式。JVM是整个计算机虚拟模型，所以JMM是隶属于JVM的。从抽象的角度来看，JMM定义了线程和主内存之间的抽象关系：线程之间的共享变量存储在主内存（Main Memory）中，每个线程都有一个私有的本地内存（Local Memory），本地内存中存储了该线程以读/写共享变量的副本。本地内存是JMM的一个抽象概念，并不真实存在。它涵盖了缓存、写缓冲区、寄存器以及其他的硬件和编译器优化。

![JMM内存模型](/intro/0033.png)

​		CPU中运行的线程从主存中拷贝共享对象obj到它的CPU缓存，然后对变量进行重新赋值。但这个变更对运行在右边CPU中的线程不可见，因为这个更改还没有flush到主存中：要解决共享对象可见性这个问题，就用到了我们的volatile。

- 原理

  当把变量声明为volatile类型后，编译器与运行时都会注意到这个变量是共享的，因此不会将该变量上的操作与其他内存操作一起重排序。volatile变量不会被缓存在寄存器或者对其他处理器不可见的地方，因此在读取volatile类型的变量时总会返回最新写入的值。声明变量是 volatile 的，JVM 保证了每次读变量都从内存中读，跳过 CPU cache 这一步。

## 实际场景		

volatile最经典的使用方法见于其在单例模式中的应用

```java
public class Singleton {
    //声明为volatile，这样其他线程初始化了singleton，其他线程会立马看到
    private volatile static Singleton singleton;  
    private Singleton (){}  
    public static Singleton getSingleton() {
     //双检锁1，volatile保证立马可以看到其他线程创建与否，也保证了后续不频繁地synchronized
    if (singleton == null) { 
        //获取类锁
        synchronized (Singleton.class) {
            //双检锁2，保证其他线程没有初始化
        if (singleton == null) {  
            //这里也使用到了volatile第二个特性，禁止重排序,保证第一个双检锁的安全
            singleton = new Singleton();  
        }  
        }  
    }  
    return singleton;  
    }  
}
```

