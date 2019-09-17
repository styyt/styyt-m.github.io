---
title: Thread.join会释放锁吗?释放的是什么锁?会死锁吗?
date: 2019-09-16 21:04:50
tags:
- 多线程
categories: 多线程
---

答案当然是**会**，问题的关键是释放的什么锁呢？，我们先来看看join方法的源码

```java
//比如我们在thread1（下面简称t1）中调用了thread2（下面简称t2）.join()方法，t2.join(0)方法
//注意方法被synchronized修饰，所以t1中获取了t2的对象锁
public final synchronized void join(long millis)
    //默认0，不超时
    throws InterruptedException {
        long base = System.currentTimeMillis();
        long now = 0;

        if (millis < 0) {
            throw new IllegalArgumentException("timeout value is negative");
        }
		//
        if (millis == 0) {
            //无限循环判断t2是否存活，否则的话就跳出循环，结束方法
            while (isAlive()) {
                //存活的话调用t2的wait方法
                //重点，这里就是释放了t2的对象锁
                //因为是在t1方法内调用此方法的，所以也就是让t1无限调用t2.wait，直到t2线程死亡
                wait(0);
            }
        } else {
            while (isAlive()) {
                //逻辑同上，只是加了个超时限制，t2存活时间超过millis，则自动跳出循环，t1线程继续往下走
                long delay = millis - now;
                if (delay <= 0) {
                    break;
                }
                wait(delay);
                now = System.currentTimeMillis() - base;
            }
        }
    }
```

所以通过上述源码，我们可以得知：t1线程内调用t2.join，是**不断获取和释放T2锁的一个过程**，并不是我之前理解的释放最开始的对象锁的

如下样例代码就会出现死锁，因为**t1一直等待t2线程死亡才会释放demo锁却，而t2会无限等待demo锁**

```java
public class JoinTest {
	public static void main(String[] args) throws InterruptedException {
		JoinDemo demo=new JoinDemo();
		Thread t2=new Thread(new Runnable() {
			@Override
			public void run() {
				System.err.println("Thread-2尝试获取锁");
				synchronized(demo){
					demo.doSomething();
					demo.doSomethingAgain();
				}
			}
		},"Thread-2");
		
		Thread t1=new Thread(new Runnable() {
			@Override
			public void run() {
				System.err.println("Thread-1尝试获取锁");
				synchronized(demo){
					demo.doSomething();
					
					try {
						t2.join();
					} catch (InterruptedException e) {
						e.printStackTrace();
					}
					
					demo.doSomethingAgain();
				}
			}
		},"Thread-1");
		
		t1.start();
		//休眠一秒。干扰线程
		try {
			Thread.sleep(1000);
		} catch (InterruptedException e) {
			// TODO Auto-generated catch block
			e.printStackTrace();
		}
		t2.start();
	}
}
class JoinDemo{
	public  void doSomething(){
		String name=Thread.currentThread().getName();
		System.err.println(name+"获得了锁");
		
		//休眠一秒。干扰线程
		try {
			Thread.sleep(1000);
		} catch (InterruptedException e) {
			// TODO Auto-generated catch block
			e.printStackTrace();
		}
		
		
	}
	
	public void doSomethingAgain(){
		String name=Thread.currentThread().getName();
		
		//休眠一秒。干扰线程
		try {
			Thread.sleep(1000);
		} catch (InterruptedException e) {
			// TODO Auto-generated catch block
			e.printStackTrace();
		}
		
		System.err.println(name+"释放了锁 Again");
	}
}
```

![出现了死锁](/intro/0026.png)