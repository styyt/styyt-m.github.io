---
title: 再从volatile第二个特性看JVM内存模型(JMM)
date: 2019-10-26 19:35:35
tags:
- volatile
- jvm
categories: jvm
---

上一篇文章我们从volatile第一个特性浅析了JMM（JVM内存模型）及其在JVM中的实现，这篇文章我们从其第二个特性继续深入分析JVM的内存模型，当然本文依然假设大家已经了解了java基础知识和[JVM的内存结构](https://styyt.github.io/2019/09/14/%E5%8D%87%E7%BA%A71.8%E5%AF%B9gc%E7%9A%84%E5%BD%B1%E5%93%8D/#jvm内存结构变化)。

先来回忆下的volatile的第二个特性

```
屏蔽指令重排序：指令重排序是编译器和处理器为了高效对程序进行优化的手段，它只能保证程序执行的结果时正确的，但是无法保证程序的操作顺序与代码顺序一致。这在单线程中不会构成问题，但是在多线程中就会出现问题。非常经典的例子是在单例方法中同时对字段加入volatile，就是为了防止指令重排序。
```

### 示例代码

老规矩，先上代码，这次还是拿单例模式中双检索来分析

```java
public class Singleton {
    //①声明为volatile，这样其他线程初始化了singleton，其他线程会立马看到
    private volatile static Singleton singleton;  
    private Singleton (){}  
    public static Singleton getSingleton() {
     //②双检锁1，volatile保证立马可以看到其他线程创建与否，也保证了后续不频繁地synchronized
    if (singleton == null) { 
        //获取类锁
        synchronized (Singleton.class) {
            //双检锁2，保证其他线程没有初始化
        if (singleton == null) {  
            //③这里也使用到了volatile第二个特性，禁止重排序,保证第一个双检锁的安全
            singleton = new Singleton();  
        }  
        }  
    }  
    return singleton;  
    }  
}
```

可以看得出，单例模式中初始化的代码为：singleton = new Singleton();  

但是这个赋值操作就像i++一样，不具备：有序性，原理后面再说。其实这句话被编译器编译为三段指令：

```java
memory=allocate();//向堆申请分配对象需要的内存空间
new Singleton();//初始化对象
singleton=memory；//将singleton指向申请的内存地址
```

但是**JMM中，允许编译器和处理器对指令进行重排序，但是重排序过程不会影响到单线程程序的执行，却会影响到多线程并发执行的正确性。** 

如果singleton没被volatile声明，以上三个步骤可能会被优化为

```java
memory=allocate();//①向堆申请分配对象需要的内存空间
singleton=memory;//②将singleton指向申请的内存地址
new Singleton();//③初始化对象
```

当然这个在单线程中肯定没问题，但是到了多线程中可能会出现以下问题

线程A获取CPU执行权，执行完了上面三个步骤的第二步

线程B争夺到CPU执行权，刚好开始执行双检锁的第一个锁即

```java
if (singleton == null) { //这里会直接返回false，因为singleton已经有指向了分配好的内存
        //获取类锁
        synchronized (Singleton.class) {
        ....
```

然后线程B返回，拿着这个没被初始化的singleton去get/set等其他操作，可想而知，程序肯定会出问题。

### 原理

- **指令重排序**

   **一般来说，处理器为了提高程序运行效率，可能会对输入代码进行优化，它不保证程序中各个语句的执行先后顺序同代码中的顺序一致，但是它会保证程序最终执行结果和代码顺序执行的结果是一致的。** 

   **编译器和处理器在重排序时，会遵守数据依赖性，编译器和处理器不会改变存在数据依赖关系的两个操作的执行顺序。** 

  比如以下代码

  ```java
  int i  = 3;    //A  
  int j   = 1;     //B  
  int k = i+j; //C  
  ```

  A和C之间存在数据依赖关系，同时B和C之间也存在数据依赖关系。因此在最终执行的指令序列中，C不能被重排序到A和B的前面（C排到A和B的前面，程序的结果将会被改变）。但A和B之间没有数据依赖关系，编译器和处理器可以重排序A和B之间的执行顺序。

  在计算机中，软件技术和硬件技术有一个共同的目标：**在不改变程序执行结果的前提下，尽可能的开发并行度。**编译器和处理器都遵从这一目标。
   这里所说的数据依赖性仅针对**单个处理器中执行的指令序列和单个线程中执行的操作**，在单线程程序中，对存在控制依赖的操作重排序，不会改变执行结果；但在**多线程程序中，**对存在控制依赖的操作重排序，可能会改变程序的执行结果。这是就需要**内存屏障来保证可见性了。**

- **内存屏障**

  先了解下简单的内存操作指令：
  
  - Store：将处理器缓存的数据刷新到内存中
  -  Load：将内存存储的数据拷贝到处理器的缓存中 
  
  结合起来看，内存屏障分为以下四个类型
  
  -  LoadLoad Barriers
  
    示例： Load1;LoadLoad;Load2 
  
    说明： 该屏障确保Load1数据的装载先于Load2及其后所有装载指令的的操作 
  
  -  StoreStore Barriers
  
    示例： Store1;StoreStore;Store2 
  
    说明： 该屏障确保Store1立刻刷新数据到内存(使其对其他处理器可见)的操作先于Store2及其后所有存储指令的操作
  
  -  LoadStore Barriers 
  
    示例： Load1;LoadStore;Store2 
  
    说明： 确保Load1的数据装载先于Store2及其后所有的存储指令刷新数据到内存的操作 
  
  -  StoreLoad Barriers 
  
    示例： Store1;StoreLoad;Load2 
  
    说明： 该屏障确保Store1立刻刷新数据到内存的操作先于Load2及其后所有装载装载指令的操作。它会使该屏障之前的所有内存访问指令**(存储指令和访问指令**)完成之后,才执行该屏障之后的内存访问指令，由此可见，StoreLoad Barriers同时具备其他三个屏障的效果，因此也称之为全能屏障（mfence），是目前大多数处理器所支持的；但是相对其他屏障，该屏障的开销相对昂贵。
  
   根据JMM规则，结合内存屏障的相关分析： 
  
  -  在每一个volatile写操作前面插入一个StoreStore屏障。这确保了在进行volatile写之前前面的所有普通的写操作都已经刷新到了内存。
  -  在每一个volatile写操作后面插入一个StoreLoad屏障。这样可以避免volatile写操作与后面可能存在的volatile读写操作发生重排序。 
  -  在每一个volatile读操作后面插入一个LoadLoad屏障。这样可以避免volatile读操作和后面普通的读操作进行重排序。 
  -  在每一个volatile读操作后面插入一个LoadStore屏障。这样可以避免volatile读操作和后面普通的写操作进行重排序。 



