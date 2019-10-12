---
title: 再度浅析jvm的四种垃圾收集器
date: 2019-10-07 12:45:10
tags:
- jvm
- gc
categories: gc
---

### 写在前面

在之前的文章中写到了[jvm的垃圾回收器（收集器）](https://styyt.github.io/2019/09/14/升级1.8对gc的影响/#垃圾回收器)，所以这里主要是根据实际的小例子来浅析各个垃圾回收器的特点

先贴出本次测试的测试代码

```java
package gc;

import java.util.ArrayList;
import java.util.HashMap;
import java.util.List;
import java.util.Map;
import java.util.Random;

public class SimpleTest {
	public static void main(String[] args) throws Exception {
		System.err.println("start...");
        List<Object> list = new ArrayList<Object>();
        try {
        	
        	while (true){
        		int sleep = new Random().nextInt(100);
        		if(System.currentTimeMillis() % 2 ==0){
        			list.clear();
        		}else{
        			for (int i = 0; i < 10000; i++) {
        				Map<String,String> map = new HashMap();
        				map.put("key_"+i, "value_" +System.currentTimeMillis() + i);
        				list.add(map);
        			}
        		}
        		System.err.println("list size="+list.size());
        		Thread.sleep(sleep);
        	}
        }catch(Exception e) {
        	System.err.println("list size="+list.size());
        	e.printStackTrace();
        }
    }
}
```

### 串行Serial收集器

#### 测试

jvm启动参数(大小自由配置)![启动参数](/intro/0028.png)

启动后console打印如下：![打印日志](/intro/0029.png)

```java
list size=10000
[GC (Allocation Failure) [DefNew: 3072K->320K(3072K), 0.0135612 secs] 5133K->4106K(9920K), 0.0136307 secs] [Times: user=0.02 sys=0.00, real=0.01 secs] 
[GC (Allocation Failure) [DefNew: 3072K->319K(3072K), 0.0102642 secs] 6858K->5764K(9920K), 0.0103687 secs] [Times: user=0.01 sys=0.00, real=0.01 secs] 
list size=20000
[GC (Allocation Failure) [DefNew: 3071K->3071K(3072K), 0.0000350 secs][Tenured: 5444K->6847K(6848K), 0.0409121 secs] 8516K->7488K(9920K), [Metaspace: 2673K->2673K(1056768K)], 0.0410759 secs] [Times: user=0.03 sys=0.00, real=0.04 secs] 
[Full GC (Allocation Failure) [Tenured: 6847K->6847K(6848K), 0.0438816 secs] 9919K->8953K(9920K), [Metaspace: 2673K->2673K(1056768K)], 0.0439903 secs] [Times: user=0.05 sys=0.00, real=0.04 secs] 
[Full GC (Allocation Failure) [Tenured: 6847K->6847K(6848K), 0.0460528 secs] 9919K->9536K(9920K), [Metaspace: 2673K->2673K(1056768K)], 0.0461242 secs] [Times: user=0.05 sys=0.00, real=0.05 secs] 
list size=30000
[Full GC (Allocation Failure) [Tenured: 6847K->6847K(6848K), 0.0448030 secs] 9919K->9833K(9920K), [Metaspace: 2673K->2673K(1056768K)], 0.0449005 secs] [Times: user=0.05 sys=0.00, real=0.05 secs] 
[Full GC (Allocation Failure) [Tenured: 6847K->6847K(6848K), 0.0295530 secs] 9919K->9885K(9920K), [Metaspace: 2673K->2673K(1056768K)], 0.0296155 secs] [Times: user=0.03 sys=0.00, real=0.03 secs] 
[Full GC (Allocation Failure) [Tenured: 6847K->6847K(6848K), 0.0267813 secs] 9919K->9906K(9920K), [Metaspace: 2673K->2673K(1056768K)], 0.0268508 secs] [Times: user=0.03 sys=0.00, real=0.03 secs] 
[Full GC (Allocation Failure) [Tenured: 6847K->6847K(6848K), 0.0289297 secs] 9919K->9914K(9920K), [Metaspace: 2673K->2673K(1056768K)], 0.0290020 secs] [Times: user=0.03 sys=0.00, real=0.03 secs] 
[Full GC (Allocation Failure) [Tenured: 6847K->6847K(6848K), 0.0296141 secs] 9919K->9917K(9920K), [Metaspace: 2673K->2673K(1056768K)], 0.0296892 secs] [Times: user=0.03 sys=0.00, real=0.03 secs] 
[Full GC (Allocation Failure) [Tenured: 6847K->6847K(6848K), 0.0190014 secs] 9919K->9919K(9920K), [Metaspace: 2673K->2673K(1056768K)], 0.0190429 secs] [Times: user=0.02 sys=0.00, real=0.02 secs] 
[Full GC (Allocation Failure) [Tenured: 6847K->6847K(6848K), 0.0231675 secs] 9919K->9919K(9920K), [Metaspace: 2673K->2673K(1056768K)], 0.0232086 secs] [Times: user=0.02 sys=0.00, real=0.02 secs] 
[Full GC (Allocation Failure) [Tenured: 6847K->6847K(6848K), 0.0181061 secs] 9919K->9919K(9920K), [Metaspace: 2673K->2673K(1056768K)], 0.0181411 secs] [Times: user=0.02 sys=0.00, real=0.02 secs] 
[Full GC (Allocation Failure) [Tenured: 6847K->6847K(6848K), 0.0197833 secs] 9919K->9919K(9920K), [Metaspace: 2673K->2673K(1056768K)], 0.0198197 secs] [Times: user=0.01 sys=0.00, real=0.02 secs] 
[Full GC (Allocation Failure) [Tenured: 6847K->6847K(6848K), 0.0168218 secs] 9919K->9919K(9920K), [Metaspace: 2673K->2673K(1056768K)], 0.0168498 secs] [Times: user=0.02 sys=0.00, real=0.02 secs] 
[Full GC (Allocation Failure) [Tenured: 6847K->507K(6848K), 0.0039987 secs] 9919K->507K(9920K), [Metaspace: 2673K->2673K(1056768K)], 0.0040402 secs] [Times: user=0.00 sys=0.00, real=0.00 secs] 
Exception in thread "main" java.lang.OutOfMemoryError: Java heap space
	at java.util.Arrays.copyOf(Arrays.java:3332)
	at java.lang.AbstractStringBuilder.ensureCapacityInternal(AbstractStringBuilder.java:124)
	at java.lang.AbstractStringBuilder.append(AbstractStringBuilder.java:674)
	at java.lang.StringBuilder.append(StringBuilder.java:208)
	at gc.SimpleTest.main(SimpleTest.java:22)
Heap
 def new generation   total 3072K, used 74K [0x00000000ff600000, 0x00000000ff950000, 0x00000000ff950000)
  eden space 2752K,   2% used [0x00000000ff600000, 0x00000000ff612848, 0x00000000ff8b0000)
  from space 320K,   0% used [0x00000000ff900000, 0x00000000ff900000, 0x00000000ff950000)
  to   space 320K,   0% used [0x00000000ff8b0000, 0x00000000ff8b0000, 0x00000000ff900000)
 tenured generation   total 6848K, used 507K [0x00000000ff950000, 0x0000000100000000, 0x0000000100000000)
   the space 6848K,   7% used [0x00000000ff950000, 0x00000000ff9cefc0, 0x00000000ff9cf000, 0x0000000100000000)
 Metaspace       used 2706K, capacity 4486K, committed 4864K, reserved 1056768K
  class space    used 285K, capacity 386K, committed 512K, reserved 1048576K

```

GC日志解读

年轻代的内存GC前后的大小：

- DefNew

​      表示使用的是串行垃圾收集器。

- 4416K->512K(4928K)

​      表示，年轻代 GC前，占有4416K内存，GC后，占有512K内存，总大小4928K

- 0.0046102 secs

​      表示， GC所用的时间，单位为毫秒。

- 4416K->1973K(15872K)

​      表示， GC前，堆内存占有4416K，GC后，占有1973K，总大小为15872K

- Full GC

​      表示，内存空间全部进行 GC

### ParNew垃圾收集器

#### 测试

jvm启动参数修改为

```java
‐XX:+UseParNewGC
```

日志打印如下

```java
list size=0
[GC (Allocation Failure) [ParNew: 3072K->3072K(3072K), 0.0000257 secs][Tenured: 5707K->1497K(6848K), 0.0071930 secs] 8779K->1497K(9920K), [Metaspace: 2658K->2658K(1056768K)], 0.0072961 secs] [Times: user=0.02 sys=0.00, real=0.01 secs] 
[GC (Allocation Failure) [ParNew: 2752K->320K(3072K), 0.0033581 secs] 4249K->2953K(9920K), 0.0034034 secs] [Times: user=0.00 sys=0.00, real=0.00 secs] 
list size=10000
list size=0
list size=0
list size=0
list size=0
[GC (Allocation Failure) [ParNew: 3072K->320K(3072K), 0.0025230 secs] 5705K->3417K(9920K), 0.0025664 secs] [Times: user=0.06 sys=0.00, real=0.00 secs] 
[GC (Allocation Failure) [ParNew: 3072K->320K(3072K), 0.0067521 secs] 6169K->4865K(9920K), 0.0068179 secs] [Times: user=0.00 sys=0.00, real=0.01 secs] 
list size=10000
[GC (Allocation Failure) [ParNew: 3072K->318K(3072K), 0.0100025 secs] 7617K->6367K(9920K), 0.0100939 secs] [Times: user=0.06 sys=0.00, real=0.01 secs] 
[GC (Allocation Failure) [ParNew: 3070K->3070K(3072K), 0.0000490 secs][Tenured: 6049K->5798K(6848K), 0.0389989 secs] 9119K->5798K(9920K), [Metaspace: 2667K->2667K(1056768K)], 0.0391808 secs] [Times: user=0.03 sys=0.00, real=0.04 secs] 
list size=20000
[GC (Allocation Failure) [ParNew: 2752K->2752K(3072K), 0.0000541 secs][Tenured: 5798K->6783K(6848K), 0.0474347 secs] 8550K->7129K(9920K), [Metaspace: 2668K->2668K(1056768K)], 0.0476134 secs] [Times: user=0.05 sys=0.00, real=0.05 secs] 
[Full GC (Allocation Failure) [Tenured: 6847K->6847K(6848K), 0.0319808 secs] 9919K->8611K(9920K), [Metaspace: 2668K->2668K(1056768K)], 0.0320699 secs] [Times: user=0.03 sys=0.00, real=0.03 secs] 
[Full GC (Allocation Failure) [Tenured: 6847K->6847K(6848K), 0.0243661 secs] 9919K->9292K(9920K), [Metaspace: 2668K->2668K(1056768K)], 0.0244230 secs] [Times: user=0.01 sys=0.00, real=0.02 secs] 
list size=30000
[Full GC (Allocation Failure) [Tenured: 6847K->6847K(6848K), 0.0467307 secs] 9919K->9627K(9920K), [Metaspace: 2668K->2668K(1056768K)], 0.0468049 secs] [Times: user=0.05 sys=0.00, real=0.05 secs] 
[Full GC (Allocation Failure) [Tenured: 6848K->6803K(6848K), 0.0364255 secs] 9920K->9771K(9920K), [Metaspace: 2668K->2668K(1056768K)], 0.0365155 secs] [Times: user=0.05 sys=0.00, real=0.04 secs] 
[Full GC (Allocation Failure) [Tenured: 6847K->6847K(6848K), 0.0293776 secs] 9919K->9882K(9920K), [Metaspace: 2668K->2668K(1056768K)], 0.0294410 secs] [Times: user=0.03 sys=0.00, real=0.03 secs] 
[Full GC (Allocation Failure) [Tenured: 6847K->6847K(6848K), 0.0303867 secs] 9919K->9901K(9920K), [Metaspace: 2668K->2668K(1056768K)], 0.0304427 secs] [Times: user=0.03 sys=0.00, real=0.03 secs] 
[Full GC (Allocation Failure) [Tenured: 6847K->6847K(6848K), 0.0338721 secs] 9919K->9911K(9920K), [Metaspace: 2668K->2668K(1056768K)], 0.0339440 secs] [Times: user=0.03 sys=0.00, real=0.03 secs] 
[Full GC (Allocation Failure) [Tenured: 6847K->6826K(6848K), 0.0218594 secs] 9919K->9894K(9920K), [Metaspace: 2668K->2668K(1056768K)], 0.0219154 secs] [Times: user=0.02 sys=0.00, real=0.02 secs] 
[Full GC (Allocation Failure) [Tenured: 6847K->6847K(6848K), 0.0154865 secs] 9919K->9917K(9920K), [Metaspace: 2668K->2668K(1056768K)], 0.0155295 secs] [Times: user=0.02 sys=0.00, real=0.02 secs] 
[Full GC (Allocation Failure) [Tenured: 6847K->6847K(6848K), 0.0166426 secs] 9919K->9918K(9920K), [Metaspace: 2668K->2668K(1056768K)], 0.0166897 secs] [Times: user=0.01 sys=0.00, real=0.02 secs] 
[Full GC (Allocation Failure) [Tenured: 6847K->6847K(6848K), 0.0159736 secs] 9919K->9919K(9920K), [Metaspace: 2668K->2668K(1056768K)], 0.0160268 secs] [Times: user=0.02 sys=0.00, real=0.02 secs] 
[Full GC (Allocation Failure) [Tenured: 6847K->6837K(6848K), 0.0173382 secs] 9919K->9909K(9920K), [Metaspace: 2668K->2668K(1056768K)], 0.0173816 secs] [Times: user=0.02 sys=0.00, real=0.02 secs] 
[Full GC (Allocation Failure) [Tenured: 6847K->6847K(6848K), 0.0165157 secs] 9919K->9919K(9920K), [Metaspace: 2668K->2668K(1056768K)], 0.0165577 secs] [Times: user=0.02 sys=0.00, real=0.02 secs] 
.............
[Full GC (Allocation Failure) [Tenured: 6847K->6847K(6848K), 0.0158630 secs] 9919K->9919K(9920K), [Metaspace: 2668K->2668K(1056768K)], 0.0159041 secs] [Times: user=0.01 sys=0.00, real=0.02 secs] 
[Full GC (Allocation Failure) [Tenured: 6847K->6847K(6848K), 0.0211750 secs] 9919K->9919K(9920K), [Metaspace: 2668K->2668K(1056768K)], 0.0212109 secs] [Times: user=0.03 sys=0.00, real=0.02 secs] 
[Full GC (Allocation Failure) [Tenured: 6847K->507K(6848K), 0.0042273 secs] 9919K->507K(9920K), [Metaspace: 2668K->2668K(1056768K)], 0.0043677 secs] [Times: user=0.00 sys=0.00, real=0.00 secs] 
Exception in thread "main" java.lang.OutOfMemoryError: Java heap space
	at java.lang.AbstractStringBuilder.<init>(AbstractStringBuilder.java:68)
	at java.lang.StringBuilder.<init>(StringBuilder.java:112)
	at gc.SimpleTest.main(SimpleTest.java:22)
Heap
 par new generation   total 3072K, used 61K [0x00000000ff600000, 0x00000000ff950000, 0x00000000ff950000)
  eden space 2752K,   2% used [0x00000000ff600000, 0x00000000ff60f6e8, 0x00000000ff8b0000)
  from space 320K,   0% used [0x00000000ff900000, 0x00000000ff900000, 0x00000000ff950000)
  to   space 320K,   0% used [0x00000000ff8b0000, 0x00000000ff8b0000, 0x00000000ff900000)
 tenured generation   total 6848K, used 507K [0x00000000ff950000, 0x0000000100000000, 0x0000000100000000)
   the space 6848K,   7% used [0x00000000ff950000, 0x00000000ff9ceef0, 0x00000000ff9cf000, 0x0000000100000000)
 Metaspace       used 2699K, capacity 4486K, committed 4864K, reserved 1056768K
  class space    used 285K, capacity 386K, committed 512K, reserved 1048576K
Java HotSpot(TM) 64-Bit Server VM warning: Using the ParNew young collector with the Serial old collector is deprecated and will likely be removed in a future release
```

由以上信息可以看出， ParNew: 使用的是ParNew收集器。其他信息和串行收集器一致。

### ParallelGC垃圾收集器

ParallelGC 收集器工作机制和ParNewGC收集器一样，只是在此基础之上，新增了两个和系统吞吐量相关的参数，使得其**使用起来更加的灵活和高效。相关参数如下：**

- -XX:+UseParallelGC

年轻代使用 ParallelGC垃圾回收器，老年代使用串行回收器。

- -XX:+UseParallelOldGC

年轻代使用 ParallelGC垃圾回收器，老年代使用ParallelOldGC垃圾回收器。

- -XX:MaxGCPauseMillis

设置最大的垃圾收集时的停顿时间，单位为毫秒。需要注意的时， ParallelGC为了达到设置的停顿时间，可能会调整堆大小或其他的参数，如果堆的大小设置的较小，就会导致GC工作变得很频繁，反而可能会影响到性能。该参数使用需谨慎。

- -XX:GCTimeRatio

设置垃圾回收时间占程序运行时间的百分比，公式为 1/(1+n)。它的值为 0~100之间的数字，默认值为99，也就是垃圾回收时间不能超过1%。

- -XX:UseAdaptiveSizePolicy

自适应 GC模式，垃圾回收器将自动调整年轻代、老年代等参数，达到吞吐量、堆大小、停顿时间之间的平衡。一般用于，手动调整参数比较困难的场景，让收集器自动进行调整。

#### 测试

参数配置为

```
-XX:+UseParallelGC -XX:+UseParallelOldGC -XX:MaxGCPauseMillis=100 -XX:+PrintGCDetails -Xms10m -Xmx10m
```

日志打印为

```java
start...
list size=0
[GC (Allocation Failure) [PSYoungGen: 2048K->504K(2560K)] 2048K->1272K(9728K), 0.0019664 secs] [Times: user=0.00 sys=0.00, real=0.00 secs] 
[GC (Allocation Failure) [PSYoungGen: 2552K->504K(2560K)] 3320K->2336K(9728K), 0.0027250 secs] [Times: user=0.06 sys=0.00, real=0.00 secs] 
[GC (Allocation Failure) [PSYoungGen: 2548K->492K(2560K)] 4380K->3396K(9728K), 0.0023355 secs] [Times: user=0.00 sys=0.00, real=0.00 secs] 
list size=10000
[GC (Allocation Failure) [PSYoungGen: 2540K->494K(2560K)] 5444K->4542K(9728K), 0.0045911 secs] [Times: user=0.00 sys=0.00, real=0.00 secs] 
[GC (Allocation Failure) [PSYoungGen: 2542K->490K(2560K)] 6590K->5598K(9728K), 0.0119535 secs] [Times: user=0.05 sys=0.00, real=0.01 secs] 
list size=20000
[GC (Allocation Failure) [PSYoungGen: 2538K->506K(1536K)] 7646K->6686K(8704K), 0.0049564 secs] [Times: user=0.00 sys=0.00, real=0.00 secs] 
[Full GC (Ergonomics) [PSYoungGen: 506K->0K(1536K)] [ParOldGen: 6180K->6571K(7168K)] 6686K->6571K(8704K), [Metaspace: 2658K->2658K(1056768K)], 0.0939727 secs] [Times: user=0.34 sys=0.00, real=0.09 secs] 
[Full GC (Ergonomics) [PSYoungGen: 1024K->0K(1536K)] [ParOldGen: 6571K->7131K(7168K)] 7595K->7131K(8704K), [Metaspace: 2658K->2658K(1056768K)], 0.0177950 secs] [Times: user=0.06 sys=0.00, real=0.02 secs] 
[Full GC (Ergonomics) [PSYoungGen: 1024K->532K(1536K)] [ParOldGen: 7131K->7131K(7168K)] 8155K->7664K(8704K), [Metaspace: 2658K->2658K(1056768K)], 0.0180539 secs] [Times: user=0.06 sys=0.00, real=0.02 secs] 
[Full GC (Ergonomics) [PSYoungGen: 1024K->788K(1536K)] [ParOldGen: 7131K->7131K(7168K)] 8155K->7919K(8704K), [Metaspace: 2658K->2658K(1056768K)], 0.0195715 secs] [Times: user=0.11 sys=0.00, real=0.02 secs] 
[Full GC (Ergonomics) [PSYoungGen: 1024K->910K(1536K)] [ParOldGen: 7131K->7131K(7168K)] 8155K->8042K(8704K), [Metaspace: 2658K->2658K(1056768K)], 0.0194927 secs] [Times: user=0.02 sys=0.00, real=0.02 secs] 
[Full GC (Ergonomics) [PSYoungGen: 1024K->969K(1536K)] [ParOldGen: 7131K->7131K(7168K)] 8155K->8101K(8704K), [Metaspace: 2658K->2658K(1056768K)], 0.0173046 secs] [Times: user=0.06 sys=0.00, real=0.02 secs] 
[Full GC (Ergonomics) [PSYoungGen: 1024K->997K(1536K)] [ParOldGen: 7131K->7131K(7168K)] 8155K->8129K(8704K), [Metaspace: 2658K->2658K(1056768K)], 0.0188894 secs] [Times: user=0.06 sys=0.00, real=0.02 secs] 
[Full GC (Ergonomics) [PSYoungGen: 1024K->1011K(1536K)] [ParOldGen: 7131K->7131K(7168K)] 8155K->8143K(8704K), [Metaspace: 2658K->2658K(1056768K)], 0.0179830 secs] [Times: user=0.06 sys=0.00, real=0.02 secs] 
[Full GC (Ergonomics) [PSYoungGen: 1024K->1017K(1536K)] [ParOldGen: 7131K->7131K(7168K)] 8155K->8149K(8704K), [Metaspace: 2658K->2658K(1056768K)], 0.0175692 secs] [Times: user=0.13 sys=0.00, real=0.02 secs] 
[Full GC (Ergonomics) [PSYoungGen: 1024K->1021K(1536K)] [ParOldGen: 7131K->7131K(7168K)] 8155K->8152K(8704K), [Metaspace: 2658K->2658K(1056768K)], 0.0189268 secs] [Times: user=0.05 sys=0.00, real=0.02 secs] 
[Full GC (Ergonomics) [PSYoungGen: 1024K->1022K(1536K)] [ParOldGen: 7131K->7131K(7168K)] 8155K->8154K(8704K), [Metaspace: 2658K->2658K(1056768K)], 0.0195244 secs] [Times: user=0.06 sys=0.00, real=0.02 secs] 
[Full GC (Ergonomics) [PSYoungGen: 1024K->1023K(1536K)] [ParOldGen: 7131K->7131K(7168K)] 8155K->8154K(8704K), [Metaspace: 2658K->2658K(1056768K)], 0.0184210 secs] [Times: user=0.06 sys=0.00, real=0.02 secs] 
[Full GC (Ergonomics) [PSYoungGen: 1024K->1023K(1536K)] [ParOldGen: 7131K->7131K(7168K)] 8155K->8155K(8704K), [Metaspace: 2658K->2658K(1056768K)], 0.0184108 secs] [Times: user=0.06 sys=0.00, real=0.02 secs] 
[Full GC (Ergonomics) [PSYoungGen: 1024K->1023K(1536K)] [ParOldGen: 7131K->7131K(7168K)] 8155K->8155K(8704K), [Metaspace: 2658K->2658K(1056768K)], 0.0177912 secs] [Times: user=0.13 sys=0.00, real=0.02 secs] 
[Full GC (Ergonomics) [PSYoungGen: 1024K->1023K(1536K)] [ParOldGen: 7131K->7131K(7168K)] 8155K->8155K(8704K), [Metaspace: 2658K->2658K(1056768K)], 0.0187630 secs] [Times: user=0.06 sys=0.00, real=0.02 secs] 
[Full GC (Ergonomics) [PSYoungGen: 1024K->1023K(1536K)] [ParOldGen: 7131K->7131K(7168K)] 8155K->8155K(8704K), [Metaspace: 2658K->2658K(1056768K)], 0.0213523 secs] [Times: user=0.05 sys=0.00, real=0.02 secs] 
[Full GC (Ergonomics) [PSYoungGen: 1024K->1023K(1536K)] [ParOldGen: 7134K->7133K(7168K)] 8158K->8157K(8704K), [Metaspace: 2658K->2658K(1056768K)], 0.0187089 secs] [Times: user=0.06 sys=0.00, real=0.02 secs] 
[Full GC (Ergonomics) [PSYoungGen: 1023K->1023K(1536K)] [ParOldGen: 7133K->7133K(7168K)] 8157K->8157K(8704K), [Metaspace: 2658K->2658K(1056768K)], 0.0172883 secs] [Times: user=0.13 sys=0.00, real=0.02 secs] 
[Full GC (Ergonomics) [PSYoungGen: 1024K->1023K(1536K)] [ParOldGen: 7136K->7134K(7168K)] 8160K->8158K(8704K), [Metaspace: 2658K->2658K(1056768K)], 0.0176494 secs] [Times: user=0.06 sys=0.00, real=0.02 secs] 
[Full GC (Ergonomics) [PSYoungGen: 1024K->1024K(1536K)] [ParOldGen: 7134K->7134K(7168K)] 8158K->8158K(8704K), [Metaspace: 2658K->2658K(1056768K)], 0.0167364 secs] [Times: user=0.06 sys=0.00, real=0.02 secs] 
[Full GC (Ergonomics) [PSYoungGen: 1024K->1023K(1536K)] [ParOldGen: 7138K->7136K(7168K)] 8162K->8160K(8704K), [Metaspace: 2658K->2658K(1056768K)], 0.0182517 secs] [Times: user=0.06 sys=0.00, real=0.02 secs] 
[Full GC (Ergonomics) [PSYoungGen: 1024K->1023K(1536K)] [ParOldGen: 7139K->7138K(7168K)] 8163K->8162K(8704K), [Metaspace: 2658K->2658K(1056768K)], 0.0220208 secs] [Times: user=0.05 sys=0.00, real=0.02 secs] 
.....................................................
[Full GC (Allocation Failure) [PSYoungGen: 1024K->1024K(1536K)] [ParOldGen: 7168K->7168K(7168K)] 8192K->8192K(8704K), [Metaspace: 2658K->2658K(1056768K)], 0.0153914 secs] [Times: user=0.06 sys=0.00, real=0.01 secs] 
[Full GC (Ergonomics) [PSYoungGen: 1024K->0K(1536K)] [ParOldGen: 7168K->507K(7168K)] 8192K->507K(8704K), [Metaspace: 2658K->2658K(1056768K)], 0.0058135 secs] [Times: user=0.06 sys=0.00, real=0.01 secs] 
Exception in thread "main" java.lang.OutOfMemoryError: Java heap space
	at java.util.Arrays.copyOf(Arrays.java:3332)
	at java.lang.AbstractStringBuilder.ensureCapacityInternal(AbstractStringBuilder.java:124)
	at java.lang.AbstractStringBuilder.append(AbstractStringBuilder.java:674)
	at java.lang.StringBuilder.append(StringBuilder.java:208)
	at gc.SimpleTest.main(SimpleTest.java:22)
Heap
 PSYoungGen      total 1536K, used 31K [0x00000000ffd00000, 0x0000000100000000, 0x0000000100000000)
  eden space 1024K, 3% used [0x00000000ffd00000,0x00000000ffd07d48,0x00000000ffe00000)
  from space 512K, 0% used [0x00000000fff80000,0x00000000fff80000,0x0000000100000000)
  to   space 1024K, 0% used [0x00000000ffe00000,0x00000000ffe00000,0x00000000fff00000)
 ParOldGen       total 7168K, used 507K [0x00000000ff600000, 0x00000000ffd00000, 0x00000000ffd00000)
  object space 7168K, 7% used [0x00000000ff600000,0x00000000ff67eec0,0x00000000ffd00000)
 Metaspace       used 2689K, capacity 4486K, committed 4864K, reserved 1056768K
  class space    used 285K, capacity 386K, committed 512K, reserved 1048576K
```

以上信息可以看出，年轻代和老年代都使用了ParallelGC垃圾回收器。

### CMS垃圾收集器

CMS全称 Concurrent Mark Sweep，是一款并发的、使用标记-清除算法的垃圾回收器，该回收器是针对老年代垃圾回收的，通过参数-XX:+UseConcMarkSweepGC进行设置。
**CMS垃圾回收器的执行过程如下：**

- 初始化标记 (CMS-initial-mark) ,标记root，会导致stw；
- 并发标记 (CMS-concurrent-mark)，与用户线程同时运行；
- 预清理（ CMS-concurrent-preclean），与用户线程同时运行；
- 重新标记 (CMS-remark) ，会导致stw；
- 并发清除 (CMS-concurrent-sweep)，与用户线程同时运行；
- 调整堆大小，设置 CMS在清理之后进行内存压缩，目的是清理内存中的碎片；
- 并发重置状态等待下次 CMS的触发(CMS-concurrent-reset)，与用户线程同时运行；

#### 测试

参数设置如下

```
‐XX:+UseConcMarkSweepGC
```

日志打印如下

```java
start...
list size=0
[GC (Allocation Failure) [ParNew: 1088K->128K(1216K), 0.0035307 secs] 1088K->755K(6016K), 0.0036180 secs] [Times: user=0.00 sys=0.00, real=0.00 secs] 
[GC (Allocation Failure) [ParNew: 1216K->126K(1216K), 0.0020168 secs] 1843K->1327K(6016K), 0.0020728 secs] [Times: user=0.00 sys=0.00, real=0.00 secs] 
[GC (Allocation Failure) [ParNew: 1214K->126K(1216K), 0.0024264 secs] 2415K->1885K(6016K), 0.0024904 secs] [Times: user=0.00 sys=0.00, real=0.00 secs] 
[GC (Allocation Failure) [ParNew: 1214K->128K(1216K), 0.0028846 secs] 2973K->2442K(6016K), 0.0029378 secs] [Times: user=0.00 sys=0.00, real=0.00 secs] 
[GC (Allocation Failure) [ParNew: 1216K->128K(1216K), 0.0022827 secs] 3530K->3006K(6016K), 0.0023443 secs] [Times: user=0.00 sys=0.00, real=0.00 secs] 
[GC (CMS Initial Mark) [1 CMS-initial-mark: 2878K(4800K)] 3006K(6016K), 0.0002025 secs] [Times: user=0.00 sys=0.00, real=0.00 secs] 
[CMS-concurrent-mark-start]
[CMS-concurrent-mark: 0.004/0.004 secs] [Times: user=0.00 sys=0.00, real=0.01 secs] 
[CMS-concurrent-preclean-start]
[CMS-concurrent-preclean: 0.000/0.000 secs] [Times: user=0.00 sys=0.00, real=0.00 secs] 
[GC (CMS Final Remark) [YG occupancy: 1075 K (1216 K)][Rescan (parallel) , 0.0008869 secs][weak refs processing, 0.0000145 secs][class unloading, 0.0003522 secs][scrub symbol table, 0.0004824 secs][scrub string table, 0.0001484 secs][1 CMS-remark: 2878K(4800K)] 3953K(6016K), 0.0020089 secs] [Times: user=0.00 sys=0.00, real=0.00 secs] 
[CMS-concurrent-sweep-start]
[GC (Allocation Failure) [ParNew: 1216K->128K(1216K), 0.0016977 secs] 4090K->3593K(6016K), 0.0017346 secs] [Times: user=0.00 sys=0.00, real=0.00 secs] 
list size=10000
[CMS-concurrent-sweep: 0.002/0.003 secs] [Times: user=0.00 sys=0.00, real=0.00 secs] 
[CMS-concurrent-reset-start]
[CMS-concurrent-reset: 0.000/0.000 secs] [Times: user=0.00 sys=0.00, real=0.00 secs] 
[GC (Allocation Failure) [ParNew: 1216K->126K(1216K), 0.0050824 secs] 4681K->4215K(6016K), 0.0051738 secs] [Times: user=0.06 sys=0.00, real=0.01 secs] 
[GC (CMS Initial Mark) [1 CMS-initial-mark: 4089K(4800K)] 4237K(6016K), 0.0003872 secs] [Times: user=0.00 sys=0.00, real=0.00 secs] 
[CMS-concurrent-mark-start]
[GC (Allocation Failure) [ParNew (promotion failed): 1214K->1216K(1216K), 0.0098695 secs][CMS[CMS-concurrent-mark: 0.016/0.027 secs] [Times: user=0.06 sys=0.00, real=0.03 secs] 
 (concurrent mode failure): 4591K->4734K(4800K), 0.0279952 secs] 5303K->4734K(6016K), [Metaspace: 2658K->2658K(1056768K)], 0.0380112 secs] [Times: user=0.09 sys=0.00, real=0.04 secs] 
[GC (Allocation Failure) [ParNew: 1088K->1088K(1216K), 0.0000336 secs][CMS: 4734K->4799K(4800K), 0.0198678 secs] 5822K->5284K(6016K), [Metaspace: 2658K->2658K(1056768K)], 0.0200119 secs] [Times: user=0.02 sys=0.00, real=0.02 secs] 
[GC (CMS Initial Mark) [1 CMS-initial-mark: 4799K(4800K)] 5306K(6016K), 0.0008309 secs] [Times: user=0.00 sys=0.00, real=0.00 secs] 
[CMS-concurrent-mark-start]
[Full GC (Allocation Failure) [CMS[CMS-concurrent-mark: 0.007/0.007 secs] [Times: user=0.03 sys=0.00, real=0.01 secs] 
 (concurrent mode failure): 4799K->4799K(4800K), 0.0274470 secs] 6015K->5651K(6016K), [Metaspace: 2658K->2658K(1056768K)], 0.0275166 secs] [Times: user=0.02 sys=0.00, real=0.03 secs] 
[Full GC (Allocation Failure) [CMS: 4799K->4799K(4800K), 0.0180525 secs] 6015K->5840K(6016K), [Metaspace: 2658K->2658K(1056768K)], 0.0181085 secs] [Times: user=0.03 sys=0.00, real=0.02 secs] 
[GC (CMS Initial Mark) [1 CMS-initial-mark: 4799K(4800K)] 5855K(6016K), 0.0011738 secs] [Times: user=0.00 sys=0.00, real=0.00 secs] 
[CMS-concurrent-mark-start]
[Full GC (Allocation Failure) [CMS[CMS-concurrent-mark: 0.007/0.008 secs] [Times: user=0.00 sys=0.00, real=0.01 secs] 
 (concurrent mode failure): 4799K->4799K(4800K), 0.0176027 secs] 6015K->5931K(6016K), [Metaspace: 2658K->2658K(1056768K)], 0.0176634 secs] [Times: user=0.02 sys=0.00, real=0.02 secs] 
[Full GC (Allocation Failure) [CMS: 4799K->4799K(4800K), 0.0119344 secs] 6015K->5975K(6016K), [Metaspace: 2658K->2658K(1056768K)], 0.0119885 secs] [Times: user=0.02 sys=0.00, real=0.01 secs] 
[GC (CMS Initial Mark) [1 CMS-initial-mark: 4799K(4800K)] 5975K(6016K), 0.0010040 secs] [Times: user=0.00 sys=0.00, real=0.00 secs] 
[CMS-concurrent-mark-start]
[Full GC (Allocation Failure) [CMS[CMS-concurrent-mark: 0.005/0.005 secs] [Times: user=0.00 sys=0.00, real=0.00 secs] 
 (concurrent mode failure): 4799K->4799K(4800K), 0.0163039 secs] 6015K->5996K(6016K), [Metaspace: 2658K->2658K(1056768K)], 0.0163422 secs] [Times: user=0.02 sys=0.00, real=0.02 secs] 
[Full GC (Allocation Failure) [CMS: 4799K->4799K(4800K), 0.0125017 secs] 6015K->6006K(6016K), [Metaspace: 2658K->2658K(1056768K)], 0.0125385 secs] [Times: user=0.02 sys=0.00, real=0.01 secs] 
[GC (CMS Initial Mark) [1 CMS-initial-mark: 4799K(4800K)] 6006K(6016K), 0.0011500 secs] [Times: user=0.00 sys=0.00, real=0.00 secs] 
[CMS-concurrent-mark-start]
[Full GC (Allocation Failure) [CMS[CMS-concurrent-mark: 0.004/0.004 secs] [Times: user=0.00 sys=0.00, real=0.00 secs] 
 (concurrent mode failure): 4800K->4799K(4800K), 0.0148129 secs] 6015K->6011K(6016K), [Metaspace: 2658K->2658K(1056768K)], 0.0148465 secs] [Times: user=0.01 sys=0.00, real=0.02 secs] 
[Full GC (Allocation Failure) [CMS: 4799K->4799K(4800K), 0.0121676 secs] 6015K->6013K(6016K), [Metaspace: 2658K->2658K(1056768K)], 0.0122036 secs] [Times: user=0.02 sys=0.00, real=0.01 secs] 
[GC (CMS Initial Mark) [1 CMS-initial-mark: 4799K(4800K)] 6013K(6016K), 0.0010936 secs] [Times: user=0.00 sys=0.00, real=0.00 secs] 
[CMS-concurrent-mark-start]
[Full GC (Allocation Failure) [CMS[CMS-concurrent-mark: 0.004/0.004 secs] [Times: user=0.00 sys=0.00, real=0.00 secs] 
 (concurrent mode failure): 4799K->4799K(4800K), 0.0157077 secs] 6015K->6014K(6016K), [Metaspace: 2658K->2658K(1056768K)], 0.0157595 secs] [Times: user=0.01 sys=0.00, real=0.02 secs] 
[Full GC (Allocation Failure) [CMS: 4799K->4799K(4800K), 0.0124317 secs] 6015K->6015K(6016K), [Metaspace: 2658K->2658K(1056768K)], 0.0124886 secs] [Times: user=0.02 sys=0.00, real=0.01 secs] 
[GC (CMS Initial Mark) [1 CMS-initial-mark: 4799K(4800K)] 6015K(6016K), 0.0013837 secs] [Times: user=0.00 sys=0.00, real=0.00 secs] 
[CMS-concurrent-mark-start]
[Full GC (Allocation Failure) [CMS[CMS-concurrent-mark: 0.005/0.005 secs] [Times: user=0.00 sys=0.00, real=0.01 secs] 
 (concurrent mode failure): 4799K->4799K(4800K), 0.0170466 secs] 6015K->6015K(6016K), [Metaspace: 2658K->2658K(1056768K)], 0.0170802 secs] [Times: user=0.02 sys=0.00, real=0.02 secs] 
[Full GC (Allocation Failure) [CMS: 4799K->4799K(4800K), 0.0103463 secs] 6015K->6015K(6016K), [Metaspace: 2658K->2658K(1056768K)], 0.0103845 secs] [Times: user=0.00 sys=0.00, real=0.01 secs] 
[GC (CMS Initial Mark) [1 CMS-initial-mark: 4799K(4800K)] 6015K(6016K), 0.0011253 secs] [Times: user=0.00 sys=0.00, real=0.00 secs] 
[CMS-concurrent-mark-start]
[Full GC (Allocation Failure) [CMS[CMS-concurrent-mark: 0.005/0.005 secs] [Times: user=0.00 sys=0.03, real=0.01 secs] 
 (concurrent mode failure): 4799K->4799K(4800K), 0.0155999 secs] 6015K->6015K(6016K), [Metaspace: 2658K->2658K(1056768K)], 0.0156512 secs] [Times: user=0.02 sys=0.00, real=0.02 secs] 
[Full GC (Allocation Failure) [CMS: 4799K->4799K(4800K), 0.0122106 secs] 6015K->6015K(6016K), [Metaspace: 2658K->2658K(1056768K)], 0.0122493 secs] [Times: user=0.00 sys=0.00, real=0.01 secs] 
[Full GC (Allocation Failure) [CMS: 4799K->512K(4800K), 0.0036926 secs] 6015K->512K(6016K), [Metaspace: 2659K->2659K(1056768K)], 0.0037267 secs] [Times: user=0.02 sys=0.00, real=0.00 secs] 
Exception in thread "main" java.lang.OutOfMemoryError: Java heap space
	at java.util.Arrays.copyOf(Arrays.java:3332)
	at java.lang.AbstractStringBuilder.ensureCapacityInternal(AbstractStringBuilder.java:124)
	at java.lang.AbstractStringBuilder.append(AbstractStringBuilder.java:674)
	at java.lang.StringBuilder.append(StringBuilder.java:208)
	at gc.SimpleTest.main(SimpleTest.java:22)
Heap
 par new generation   total 1216K, used 36K [0x00000000ffa00000, 0x00000000ffb50000, 0x00000000ffb50000)
  eden space 1088K,   3% used [0x00000000ffa00000, 0x00000000ffa09208, 0x00000000ffb10000)
  from space 128K,   0% used [0x00000000ffb10000, 0x00000000ffb10000, 0x00000000ffb30000)
  to   space 128K,   0% used [0x00000000ffb30000, 0x00000000ffb30000, 0x00000000ffb50000)
 concurrent mark-sweep generation total 4800K, used 512K [0x00000000ffb50000, 0x0000000100000000, 0x0000000100000000)
 Metaspace       used 2689K, capacity 4486K, committed 4864K, reserved 1056768K
  class space    used 285K, capacity 386K, committed 512K, reserved 1048576K
```

根据上述信息可以看到，老年代使用的是CMS，新生代使用的还是Parnew

```java
#第一步，初始标记
[GC (CMS Initial Mark) [1 CMS‐initial‐mark: 6224K(10944K)] 6824K(15872K),
0.0004209 secs] [Times: user=0.00 sys=0.00, real=0.00 secs]

#第二步，并发标记
[CMS‐concurrent‐mark‐start]
[CMS‐concurrent‐mark: 0.002/0.002 secs] [Times: user=0.00 sys=0.00,
real=0.00 secs]

#第三步，预处理
[CMS‐concurrent‐preclean‐start]
[CMS‐concurrent‐preclean: 0.000/0.000 secs] [Times: user=0.00 sys=0.00,
real=0.00 secs]
```

### G1(Garbage First)垃圾收集器（重要）

G1垃圾收集器是在jdk1.7中正式使用的全新的垃圾收集器，oracle官方计划在jdk9中将G1变成默认的垃圾收集器，以替代CMS。

G1 的设计原则就是简化JVM性能调优，开发人员只需要简单的三步即可完成调优：

- 第一步，开启G1垃圾收集器
- 第二步，设置堆的最大内存
- 第三步，设置最大的停顿时间

即声明以下参数即可：

-XX:+UseG1GC -Xmx32g -XX:MaxGCPauseMillis=200

其中-XX:+UseG1GC为开启G1垃圾收集器，-Xmx32g 设计堆内存的最大内存为32G，-XX:MaxGCPauseMillis=200设置GC的最大暂停时间为200ms。如果我们需要调优，在内存大小一定的情况下，我们只需要修改最大暂停时间即可。

其次，G1将新生代，老年代的物理空间划分取消了。

这样我们再也不用单独的空间对每个代进行设置了，不用担心每个代内存是否足够。

#### 原理



##### G1收集器内存划分方式

![G1划分](/intro/0030.png)

G1算法将堆划分为若干个区域（Region），它仍然属于分代收集器。不过，这些区域的一部分包含新生代，新生代的垃圾收集依然采用暂停所有应用线程的方式，将存活对象拷贝到老年代或者Survivor空间。老年代也分成很多区域，G1收集器通过将对象从一个区域复制到另外一个区域，完成了清理工作。这就意味着，在正常的处理过程中，G1完成了堆的压缩（至少是部分堆的压缩），这样也就不会有cms内存碎片问题的存在了。

PS:在G1中，还有一种特殊的区域，叫Humongous区域。 如果一个对象占用的空间超过了分区容量50%以上，G1收集器就认为这是一个巨型对象。这些巨型对象，默认直接会被分配在年老代，但是如果它是一个短期存在的巨型对象，就会对垃圾收集器造成负面影响。为了解决这个问题，G1划分了一个Humongous区，它用来专门存放巨型对象。如果一个H区装不下一个巨型对象，那么G1会寻找连续的H分区来存储。为了能找到连续的H区，有时候不得不启动Full GC。

G1以一种和CMS 相似的方式执行垃圾回收。G1执行通过在并发标记阶段决定heap内对象存活性。标记阶段后，G1 知道哪些regions 是更空的，然后它优先处理垃圾多的region。这也是为啥被叫做Garbage-First的原因。如名字一样，G1会集中在对那些可能全部被回收的堆空间。G1通过用停顿预测模型来满足用户自定义的停顿时间目标，它基于设定的停顿时间来选择要回收的regions数量。

G1清理那些被确认为成熟的可以清理的regions,通过从一个或多个region里，copy对象到另一个region里，中间伴随着压缩和清理内存的工作。清理过程在多核机器上都采用并行执行，来降低停顿时间，增加吞吐量。因此 G1在持续的运行中 能减少碎片，满足用户自定义停顿时间需求。这种能力是以往的回收器所不具备的。CMS回收器不能进行碎片压缩，ParallelOld 只能进行整堆的压缩，会导致较长的停顿时间。

值得注意的是：G1不是一个实时的收集器，它只是最大可能的来满足设定的停顿时间。G1会基于以往的收集数据，来评估用户指定的停顿时间可以回收多少regions。G1要通过模型评估出要收集的regions需要花费的时间，然后决定停顿时间内可以回收多少个regions。这就是G1一个及其重要的特性：软实时（soft real-time）。所谓的实时垃圾回收，是指在要求的时间内完成垃圾回收。“软实时”则是指，用户可以指定垃圾回收时间的限时，G1会努力在这个时限内完成垃圾回收，但是G1**并不担保每次都能在这个时限内完成垃圾回收**。通过设定一个合理的目标，可以让达到90%以上的垃圾回收时间都在这个时限内。

##### G1 Footprint

如果你从ParallelOldGC 或 CMS 移到G1,你将会看到JVM将需要更大的内存。主要是因为是因为G1用到相关的统计数据结构如： Remembered Sets 和 Collection Sets。

Remembered Sets 主要是是对region内对象引用的跟踪。每个region都有一个RSet。Rset占用量小于总足迹大小的5%。

Collection Sets 是指代 将要被收集region的集合。GC过程中，CSet所有存活的数据将被整理(copied/moved)。这些region的集合可能是Eden区 surivor 或者 old区。CSet大小大概是JVM大小的1%。

##### G1适合的场景

G1主要面向的场景就是大内存，同时要求低的GC延迟场景。一般6GB大小的内存，稳定的预测停顿时间一般小于0.5秒。

现在在CMS或者ParallelOldGC上运行的程序，如果具备以下特点，迁移到G1将会获得更大的好处。

- Full GC 时间太长 或太频繁；
- 对象分配速度或者晋升比例变化明显；
- 不希望GC 或者压缩停顿时间太长；

如果你现在用CMS 或者 ParallelOldGC ，并且你的程序运行很好，没有经历长时间垃圾回收停顿，建议就不用迁移。

**G1中提供了三种模式垃圾回收模式，Young GC、Mixed GC 和 Full GC，在不同的条下被触发。**

#### Young GC

主要是对Eden区进行GC，它在Eden空间耗尽时会被触发。

- Eden 空间的数据移动到Survivor空间中，如果Survivor空间不够，Eden空间的部分数据会直接晋升到年老代空间。
- Survivor 区的数据移动到新的Survivor区中，也有部分数据晋升到老年代空间中。最终 Eden空间的数据为空，GC停止工作，应用线程继续执行。

这时，我们需要考虑一个问题，如果仅仅GC 新生代对象，我们如何找到所有的根对象呢？ 老年代的所有对象都是根么？那这样扫描下来会耗费大量的时间。于是，G1引进了RSet的概念。它的全称是Remembered Set，作用是跟踪指向某个heap区内的对象引用。

每个 Region初始化时，会初始化一个RSet，该集合用来记录并跟踪其它Region指向该Region中对象的引用，每个Region默认按照512Kb划分成多个Card，所以RSet需要记录的东西应该是 xx Region的 xx Card。

#### Mixed GC

当越来越多的对象晋升到老年代old region时，为了避免堆内存被耗尽，虚拟机会触发一个混合的垃圾收集器，即Mixed GC，该算法并不是一个Old GC，除了回收整个YoungRegion，还会回收一部分的Old Region，这里需要注意：是一部分老年代，而不是全部老年代，可以选择哪些old region进行收集，从而可以对垃圾回收的耗时时间进行控制。也要注意的是Mixed GC 并不是 Full GC。
**MixedGC什么时候触发？** 由参数 -XX:InitiatingHeapOccupancyPercent=n 决定。默
认：45%，该参数的意思是：当老年代大小占整个堆大小百分比达到该阀值时触发。它的GC步骤分2步：

- 全局并发标记（global concurrent marking）
- 拷贝存活对象（evacuation）

**A、全局并发标记**
全局并发标记，执行过程分为五个步骤：

- 1、初始标记（ initial mark，STW）

标记从根节点直接可达的对象，这个阶段会执行一次年轻代 GC，会产生全局停顿。

- 2、根区域扫描（ root region scan）

G1 GC  在初始标记的存活区扫描对老年代的引用，并标记被引用的对象。该阶段与应用程序（非 STW）同时运行，并且只有完成该阶段后，才能开始下一次 STW 年轻代垃圾回收。

- 3、并发标记（Concurrent Marking）

G1 GC  在整个堆中查找可访问的（存活的）对象。该阶段与应用程序同时运行，可以被 STW 年轻代垃圾回收中断。

- 4、重新标记（ Remark，STW）

该阶段是 STW 回收，因为程序在运行，针对上一次的标记进行修正。

- 5、清除垃圾（ Cleanup，STW）

清点和重置标记状态，该阶段会 STW，这个阶段并不会实际上去做垃圾的收集，等待evacuation阶段来回收。

**B、拷贝存活对象**

Evacuation阶段是全暂停的。该阶段把一部分Region里的活对象拷贝到另一部分Region中，从而实现垃圾的回收清理。

#### Full GC

如果对象内存分配速度过快，mixed gc来不及回收，导致老年代被填满，就会触发 一次full gc，G1的full gc算法就是单线程执行的serial old gc，会导致异常长时间的暂停时间，需要进行不断的调优，尽可能的避免full gc。

#### G1收集器相关参数

- -XX:+UseG1GC

使用 G1 垃圾收集器

- -XX:MaxGCPauseMillis

设置期望达到的最大 GC停顿时间指标（JVM会尽力实现，但不保证达到），默认值是 200 毫秒。

- -XX:G1HeapRegionSize=n

设置的 G1 区域的大小。值是 2 的幂，范围是 1 MB 到 32 MB 之间。目标是根据最小的 Java 堆大小划分出约 2048 个区域。默认是堆内存的 1/2000。

- -XX:ParallelGCThreads=n

设置 STW 工作线程数的值。将 n 的值设置为逻辑处理器的数量。n 的值与逻辑处理器的数量相同，最多为 8。

- -XX:ConcGCThreads=n

设置并行标记的线程数。将 n 设置为并行垃圾回收线程数 (ParallelGCThreads)的 1/4 左右。

- -XX:InitiatingHeapOccupancyPercent=n

设置触发标记周期的 Java 堆占用率阈值。默认占用率是整个 Java 堆的 45%。

#### 测试

日志打印如下：

```
start...
list size=0
list size=0
list size=0
[GC pause (G1 Evacuation Pause) (young), 0.0067703 secs]
   [Parallel Time: 6.0 ms, GC Workers: 4]
      [GC Worker Start (ms): Min: 393.5, Avg: 393.9, Max: 394.9, Diff: 1.4]
      [Ext Root Scanning (ms): Min: 0.0, Avg: 0.7, Max: 0.9, Diff: 0.9, Sum: 2.6]
      [Update RS (ms): Min: 0.0, Avg: 0.0, Max: 0.0, Diff: 0.0, Sum: 0.0]
         [Processed Buffers: Min: 0, Avg: 0.0, Max: 0, Diff: 0, Sum: 0]
      [Scan RS (ms): Min: 0.0, Avg: 0.0, Max: 0.0, Diff: 0.0, Sum: 0.0]
      [Code Root Scanning (ms): Min: 0.0, Avg: 0.0, Max: 0.1, Diff: 0.1, Sum: 0.1]
      [Object Copy (ms): Min: 4.0, Avg: 4.5, Max: 4.6, Diff: 0.6, Sum: 17.8]
      [Termination (ms): Min: 0.0, Avg: 0.1, Max: 0.2, Diff: 0.2, Sum: 0.6]
         [Termination Attempts: Min: 1, Avg: 1.0, Max: 1, Diff: 0, Sum: 4]
      [GC Worker Other (ms): Min: 0.1, Avg: 0.1, Max: 0.3, Diff: 0.2, Sum: 0.5]
      [GC Worker Total (ms): Min: 4.3, Avg: 5.4, Max: 5.9, Diff: 1.6, Sum: 21.6]
      [GC Worker End (ms): Min: 399.2, Avg: 399.3, Max: 399.4, Diff: 0.3]
   [Code Root Fixup: 0.0 ms]
   [Code Root Purge: 0.0 ms]
   [Clear CT: 0.1 ms]
   [Other: 0.6 ms]
      [Choose CSet: 0.0 ms]
      [Ref Proc: 0.3 ms]
      [Ref Enq: 0.0 ms]
      [Redirty Cards: 0.1 ms]
      [Humongous Register: 0.0 ms]
      [Humongous Reclaim: 0.0 ms]
      [Free CSet: 0.1 ms]
   [Eden: 2048.0K(2048.0K)->0.0B(1024.0K) Survivors: 0.0B->1024.0K Heap: 2048.0K(6144.0K)->1304.0K(6144.0K)]
 [Times: user=0.03 sys=0.03, real=0.01 secs] 
[GC pause (G1 Evacuation Pause) (young), 0.4119142 secs]
   [Parallel Time: 411.7 ms, GC Workers: 4]
      [GC Worker Start (ms): Min: 411.6, Avg: 411.9, Max: 412.6, Diff: 0.9]
      [Ext Root Scanning (ms): Min: 0.0, Avg: 0.5, Max: 0.7, Diff: 0.7, Sum: 1.8]
      [Update RS (ms): Min: 0.0, Avg: 102.5, Max: 410.0, Diff: 410.0, Sum: 410.0]
         [Processed Buffers: Min: 0, Avg: 0.5, Max: 1, Diff: 1, Sum: 2]
      [Scan RS (ms): Min: 0.0, Avg: 0.0, Max: 0.0, Diff: 0.0, Sum: 0.0]
      [Code Root Scanning (ms): Min: 0.0, Avg: 0.0, Max: 0.1, Diff: 0.1, Sum: 0.1]
      [Object Copy (ms): Min: 0.0, Avg: 5.9, Max: 8.1, Diff: 8.1, Sum: 23.8]
      [Termination (ms): Min: 0.0, Avg: 301.7, Max: 403.0, Diff: 403.0, Sum: 1207.0]
         [Termination Attempts: Min: 1, Avg: 8.8, Max: 17, Diff: 16, Sum: 35]
      [GC Worker Other (ms): Min: 0.0, Avg: 0.0, Max: 0.0, Diff: 0.0, Sum: 0.1]
      [GC Worker Total (ms): Min: 409.8, Avg: 410.7, Max: 411.6, Diff: 1.9, Sum: 1642.8]
      [GC Worker End (ms): Min: 822.3, Avg: 822.6, Max: 823.3, Diff: 1.0]
   [Code Root Fixup: 0.0 ms]
   [Code Root Purge: 0.0 ms]
   [Clear CT: 0.0 ms]
   [Other: 0.2 ms]
      [Choose CSet: 0.0 ms]
      [Ref Proc: 0.0 ms]
      [Ref Enq: 0.0 ms]
      [Redirty Cards: 0.0 ms]
      [Humongous Register: 0.0 ms]
      [Humongous Reclaim: 0.0 ms]
      [Free CSet: 0.0 ms]
   [Eden: 1024.0K(1024.0K)->0.0B(1024.0K) Survivors: 1024.0K->1024.0K Heap: 2328.0K(6144.0K)->2303.9K(6144.0K)]
 [Times: user=1.06 sys=0.00, real=0.41 secs] 
[GC pause (G1 Evacuation Pause) (young), 0.0018246 secs]
   [Parallel Time: 1.6 ms, GC Workers: 4]
      [GC Worker Start (ms): Min: 827.2, Avg: 827.2, Max: 827.2, Diff: 0.0]
      [Ext Root Scanning (ms): Min: 0.2, Avg: 0.2, Max: 0.2, Diff: 0.0, Sum: 0.7]
      [Update RS (ms): Min: 0.0, Avg: 0.1, Max: 0.2, Diff: 0.2, Sum: 0.5]
         [Processed Buffers: Min: 1, Avg: 1.0, Max: 1, Diff: 0, Sum: 4]
      [Scan RS (ms): Min: 0.0, Avg: 0.0, Max: 0.0, Diff: 0.0, Sum: 0.0]
      [Code Root Scanning (ms): Min: 0.0, Avg: 0.0, Max: 0.0, Diff: 0.0, Sum: 0.0]
      [Object Copy (ms): Min: 1.2, Avg: 1.2, Max: 1.2, Diff: 0.0, Sum: 4.7]
      [Termination (ms): Min: 0.0, Avg: 0.1, Max: 0.2, Diff: 0.2, Sum: 0.5]
         [Termination Attempts: Min: 2, Avg: 2.5, Max: 3, Diff: 1, Sum: 10]
      [GC Worker Other (ms): Min: 0.0, Avg: 0.0, Max: 0.0, Diff: 0.0, Sum: 0.1]
      [GC Worker Total (ms): Min: 1.6, Avg: 1.6, Max: 1.6, Diff: 0.0, Sum: 6.4]
      [GC Worker End (ms): Min: 828.8, Avg: 828.8, Max: 828.8, Diff: 0.0]
   [Code Root Fixup: 0.0 ms]
   [Code Root Purge: 0.0 ms]
   [Clear CT: 0.0 ms]
   [Other: 0.1 ms]
      [Choose CSet: 0.0 ms]
      [Ref Proc: 0.1 ms]
      [Ref Enq: 0.0 ms]
      [Redirty Cards: 0.0 ms]
      [Humongous Register: 0.0 ms]
      [Humongous Reclaim: 0.0 ms]
      [Free CSet: 0.0 ms]
   [Eden: 1024.0K(1024.0K)->0.0B(1024.0K) Survivors: 1024.0K->1024.0K Heap: 3327.9K(6144.0K)->2887.7K(6144.0K)]
 [Times: user=0.00 sys=0.00, real=0.00 secs] 
[GC pause (G1 Evacuation Pause) (young) (initial-mark) (to-space exhausted), 0.0031538 secs]
   [Parallel Time: 2.2 ms, GC Workers: 4]
      [GC Worker Start (ms): Min: 832.0, Avg: 832.2, Max: 832.3, Diff: 0.3]
      [Ext Root Scanning (ms): Min: 0.0, Avg: 0.2, Max: 0.3, Diff: 0.3, Sum: 0.9]
      [Update RS (ms): Min: 0.0, Avg: 0.0, Max: 0.1, Diff: 0.1, Sum: 0.1]
         [Processed Buffers: Min: 0, Avg: 0.8, Max: 2, Diff: 2, Sum: 3]
      [Scan RS (ms): Min: 0.0, Avg: 0.0, Max: 0.0, Diff: 0.0, Sum: 0.0]
      [Code Root Scanning (ms): Min: 0.0, Avg: 0.0, Max: 0.0, Diff: 0.0, Sum: 0.0]
      [Object Copy (ms): Min: 0.7, Avg: 1.4, Max: 1.7, Diff: 1.0, Sum: 5.7]
      [Termination (ms): Min: 0.0, Avg: 0.3, Max: 1.1, Diff: 1.1, Sum: 1.3]
         [Termination Attempts: Min: 1, Avg: 1.8, Max: 3, Diff: 2, Sum: 7]
      [GC Worker Other (ms): Min: 0.0, Avg: 0.0, Max: 0.0, Diff: 0.0, Sum: 0.0]
      [GC Worker Total (ms): Min: 1.9, Avg: 2.0, Max: 2.1, Diff: 0.3, Sum: 8.0]
      [GC Worker End (ms): Min: 834.2, Avg: 834.2, Max: 834.2, Diff: 0.0]
   [Code Root Fixup: 0.0 ms]
   [Code Root Purge: 0.0 ms]
   [Clear CT: 0.0 ms]
   [Other: 1.0 ms]
      [Evacuation Failure: 0.8 ms]
      [Choose CSet: 0.0 ms]
      [Ref Proc: 0.1 ms]
      [Ref Enq: 0.0 ms]
      [Redirty Cards: 0.0 ms]
      [Humongous Register: 0.0 ms]
      [Humongous Reclaim: 0.0 ms]
      [Free CSet: 0.0 ms]
   [Eden: 1024.0K(1024.0K)->0.0B(1024.0K) Survivors: 1024.0K->1024.0K Heap: 3911.7K(6144.0K)->4581.6K(6144.0K)]
 [Times: user=0.05 sys=0.02, real=0.00 secs] 
[GC concurrent-root-region-scan-start]
[GC concurrent-root-region-scan-end, 0.0015559 secs]
[GC concurrent-mark-start]
[GC pause (G1 Evacuation Pause) (young) (to-space exhausted), 0.0181089 secs]
   [Parallel Time: 16.4 ms, GC Workers: 4]
      [GC Worker Start (ms): Min: 838.4, Avg: 838.5, Max: 838.7, Diff: 0.3]
      [Ext Root Scanning (ms): Min: 0.0, Avg: 0.1, Max: 0.2, Diff: 0.2, Sum: 0.6]
      [Update RS (ms): Min: 0.0, Avg: 0.0, Max: 0.0, Diff: 0.0, Sum: 0.1]
         [Processed Buffers: Min: 0, Avg: 1.0, Max: 3, Diff: 3, Sum: 4]
      [Scan RS (ms): Min: 0.0, Avg: 0.0, Max: 0.0, Diff: 0.0, Sum: 0.0]
      [Code Root Scanning (ms): Min: 0.0, Avg: 0.0, Max: 0.0, Diff: 0.0, Sum: 0.0]
      [Object Copy (ms): Min: 0.0, Avg: 7.6, Max: 15.2, Diff: 15.2, Sum: 30.4]
      [Termination (ms): Min: 0.0, Avg: 8.0, Max: 16.1, Diff: 16.1, Sum: 32.2]
         [Termination Attempts: Min: 1, Avg: 1.0, Max: 1, Diff: 0, Sum: 4]
      [GC Worker Other (ms): Min: 0.0, Avg: 0.0, Max: 0.0, Diff: 0.0, Sum: 0.0]
      [GC Worker Total (ms): Min: 15.3, Avg: 15.8, Max: 16.4, Diff: 1.1, Sum: 63.3]
      [GC Worker End (ms): Min: 853.9, Avg: 854.4, Max: 854.8, Diff: 0.9]
   [Code Root Fixup: 0.0 ms]
   [Code Root Purge: 0.0 ms]
   [Clear CT: 0.0 ms]
   [Other: 1.7 ms]
      [Evacuation Failure: 1.5 ms]
      [Choose CSet: 0.0 ms]
      [Ref Proc: 0.0 ms]
      [Ref Enq: 0.0 ms]
      [Redirty Cards: 0.0 ms]
      [Humongous Register: 0.0 ms]
      [Humongous Reclaim: 0.0 ms]
      [Free CSet: 0.0 ms]
   [Eden: 1024.0K(1024.0K)->0.0B(1024.0K) Survivors: 1024.0K->0.0B Heap: 5605.6K(6144.0K)->5605.6K(6144.0K)]
 [Times: user=0.05 sys=0.00, real=0.02 secs] 
[Full GC (Allocation Failure)  5605K->3325K(6144K), 0.0100972 secs]
   [Eden: 0.0B(1024.0K)->0.0B(1024.0K) Survivors: 0.0B->0.0B Heap: 5605.6K(6144.0K)->3325.9K(6144.0K)], [Metaspace: 2658K->2658K(1056768K)]
 [Times: user=0.02 sys=0.00, real=0.01 secs] 
[GC concurrent-mark-abort]
list size=10000
list size=0
list size=0
list size=0
list size=0
list size=0
list size=0
[GC pause (G1 Evacuation Pause) (young), 0.0027815 secs]
   [Parallel Time: 2.3 ms, GC Workers: 4]
      [GC Worker Start (ms): Min: 1108.2, Avg: 1108.6, Max: 1109.6, Diff: 1.4]
      [Ext Root Scanning (ms): Min: 0.0, Avg: 0.4, Max: 0.5, Diff: 0.5, Sum: 1.6]
      [Update RS (ms): Min: 0.0, Avg: 0.0, Max: 0.0, Diff: 0.0, Sum: 0.1]
         [Processed Buffers: Min: 0, Avg: 0.5, Max: 1, Diff: 1, Sum: 2]
      [Scan RS (ms): Min: 0.0, Avg: 0.0, Max: 0.0, Diff: 0.0, Sum: 0.0]
      [Code Root Scanning (ms): Min: 0.0, Avg: 0.0, Max: 0.0, Diff: 0.0, Sum: 0.0]
      [Object Copy (ms): Min: 0.9, Avg: 1.5, Max: 1.7, Diff: 0.8, Sum: 6.0]
      [Termination (ms): Min: 0.0, Avg: 0.0, Max: 0.0, Diff: 0.0, Sum: 0.1]
         [Termination Attempts: Min: 4, Avg: 5.3, Max: 7, Diff: 3, Sum: 21]
      [GC Worker Other (ms): Min: 0.0, Avg: 0.0, Max: 0.0, Diff: 0.0, Sum: 0.1]
      [GC Worker Total (ms): Min: 0.9, Avg: 1.9, Max: 2.3, Diff: 1.4, Sum: 7.8]
      [GC Worker End (ms): Min: 1110.5, Avg: 1110.5, Max: 1110.5, Diff: 0.0]
   [Code Root Fixup: 0.0 ms]
   [Code Root Purge: 0.0 ms]
   [Clear CT: 0.1 ms]
   [Other: 0.4 ms]
      [Choose CSet: 0.0 ms]
      [Ref Proc: 0.1 ms]
      [Ref Enq: 0.0 ms]
      [Redirty Cards: 0.1 ms]
      [Humongous Register: 0.0 ms]
      [Humongous Reclaim: 0.0 ms]
      [Free CSet: 0.0 ms]
   [Eden: 1024.0K(1024.0K)->0.0B(1024.0K) Survivors: 0.0B->1024.0K Heap: 4349.9K(6144.0K)->3837.0K(6144.0K)]
 [Times: user=0.00 sys=0.00, real=0.00 secs] 
[GC pause (G1 Evacuation Pause) (young) (initial-mark) (to-space exhausted), 0.0386681 secs]
   [Parallel Time: 31.2 ms, GC Workers: 4]
      [GC Worker Start (ms): Min: 1120.9, Avg: 1121.3, Max: 1121.7, Diff: 0.9]
      [Ext Root Scanning (ms): Min: 0.0, Avg: 0.5, Max: 1.0, Diff: 0.9, Sum: 2.0]
      [Update RS (ms): Min: 0.0, Avg: 0.0, Max: 0.0, Diff: 0.0, Sum: 0.1]
         [Processed Buffers: Min: 0, Avg: 0.8, Max: 3, Diff: 3, Sum: 3]
      [Scan RS (ms): Min: 0.0, Avg: 0.0, Max: 0.0, Diff: 0.0, Sum: 0.0]
      [Code Root Scanning (ms): Min: 0.0, Avg: 0.0, Max: 0.0, Diff: 0.0, Sum: 0.0]
      [Object Copy (ms): Min: 29.9, Avg: 30.0, Max: 30.0, Diff: 0.1, Sum: 119.8]
      [Termination (ms): Min: 0.0, Avg: 0.1, Max: 0.1, Diff: 0.1, Sum: 0.3]
         [Termination Attempts: Min: 1, Avg: 1.0, Max: 1, Diff: 0, Sum: 4]
      [GC Worker Other (ms): Min: 0.0, Avg: 0.0, Max: 0.0, Diff: 0.0, Sum: 0.1]
      [GC Worker Total (ms): Min: 30.1, Avg: 30.6, Max: 31.0, Diff: 0.9, Sum: 122.3]
      [GC Worker End (ms): Min: 1151.8, Avg: 1151.9, Max: 1151.9, Diff: 0.0]
   [Code Root Fixup: 0.0 ms]
   [Code Root Purge: 0.0 ms]
   [Clear CT: 0.4 ms]
   [Other: 7.1 ms]
      [Evacuation Failure: 6.5 ms]
      [Choose CSet: 0.0 ms]
      [Ref Proc: 0.3 ms]
      [Ref Enq: 0.0 ms]
      [Redirty Cards: 0.1 ms]
      [Humongous Register: 0.0 ms]
      [Humongous Reclaim: 0.0 ms]
      [Free CSet: 0.0 ms]
   [Eden: 1024.0K(1024.0K)->0.0B(1024.0K) Survivors: 1024.0K->0.0B Heap: 4861.0K(6144.0K)->4861.0K(6144.0K)]
 [Times: user=0.03 sys=0.00, real=0.04 secs] 
[GC concurrent-root-region-scan-start]
[GC concurrent-root-region-scan-end, 0.0000294 secs]
[GC concurrent-mark-start]
[GC pause (G1 Evacuation Pause) (young), 0.0028533 secs]
   [Parallel Time: 2.2 ms, GC Workers: 4]
      [GC Worker Start (ms): Min: 1166.0, Avg: 1167.0, Max: 1168.0, Diff: 2.1]
      [Ext Root Scanning (ms): Min: 0.0, Avg: 0.3, Max: 0.6, Diff: 0.6, Sum: 1.2]
      [Update RS (ms): Min: 0.0, Avg: 0.6, Max: 1.4, Diff: 1.4, Sum: 2.6]
         [Processed Buffers: Min: 0, Avg: 1.3, Max: 3, Diff: 3, Sum: 5]
      [Scan RS (ms): Min: 0.0, Avg: 0.0, Max: 0.0, Diff: 0.0, Sum: 0.0]
      [Code Root Scanning (ms): Min: 0.0, Avg: 0.0, Max: 0.0, Diff: 0.0, Sum: 0.0]
      [Object Copy (ms): Min: 0.0, Avg: 0.0, Max: 0.0, Diff: 0.0, Sum: 0.1]
      [Termination (ms): Min: 0.0, Avg: 0.1, Max: 0.3, Diff: 0.3, Sum: 0.5]
         [Termination Attempts: Min: 1, Avg: 1.0, Max: 1, Diff: 0, Sum: 4]
      [GC Worker Other (ms): Min: 0.0, Avg: 0.0, Max: 0.0, Diff: 0.0, Sum: 0.1]
      [GC Worker Total (ms): Min: 0.0, Avg: 1.1, Max: 2.1, Diff: 2.1, Sum: 4.4]
      [GC Worker End (ms): Min: 1168.1, Avg: 1168.1, Max: 1168.1, Diff: 0.1]
   [Code Root Fixup: 0.0 ms]
   [Code Root Purge: 0.0 ms]
   [Clear CT: 0.1 ms]
   [Other: 0.6 ms]
      [Choose CSet: 0.0 ms]
      [Ref Proc: 0.3 ms]
      [Ref Enq: 0.0 ms]
      [Redirty Cards: 0.1 ms]
      [Humongous Register: 0.0 ms]
      [Humongous Reclaim: 0.0 ms]
      [Free CSet: 0.0 ms]
   [Eden: 0.0B(1024.0K)->0.0B(1024.0K) Survivors: 0.0B->0.0B Heap: 4861.0K(6144.0K)->4861.0K(6144.0K)]
 [Times: user=0.00 sys=0.00, real=0.00 secs] 
[Full GC (Allocation Failure)  4861K->1466K(6144K), 0.0156424 secs]
   [Eden: 0.0B(1024.0K)->0.0B(2048.0K) Survivors: 0.0B->0.0B Heap: 4861.0K(6144.0K)->1466.3K(6144.0K)], [Metaspace: 2658K->2658K(1056768K)]
 [Times: user=0.02 sys=0.00, real=0.02 secs] 
[GC concurrent-mark-abort]
[GC pause (G1 Evacuation Pause) (young), 0.0037533 secs]
   [Parallel Time: 2.9 ms, GC Workers: 4]
      [GC Worker Start (ms): Min: 1195.8, Avg: 1195.8, Max: 1195.9, Diff: 0.0]
      [Ext Root Scanning (ms): Min: 0.3, Avg: 0.3, Max: 0.3, Diff: 0.0, Sum: 1.1]
      [Update RS (ms): Min: 0.0, Avg: 0.1, Max: 0.4, Diff: 0.4, Sum: 0.4]
         [Processed Buffers: Min: 0, Avg: 0.3, Max: 1, Diff: 1, Sum: 1]
      [Scan RS (ms): Min: 0.0, Avg: 0.0, Max: 0.0, Diff: 0.0, Sum: 0.0]
      [Code Root Scanning (ms): Min: 0.0, Avg: 0.0, Max: 0.0, Diff: 0.0, Sum: 0.0]
      [Object Copy (ms): Min: 2.1, Avg: 2.4, Max: 2.5, Diff: 0.4, Sum: 9.8]
      [Termination (ms): Min: 0.0, Avg: 0.0, Max: 0.0, Diff: 0.0, Sum: 0.0]
         [Termination Attempts: Min: 1, Avg: 1.0, Max: 1, Diff: 0, Sum: 4]
      [GC Worker Other (ms): Min: 0.0, Avg: 0.0, Max: 0.0, Diff: 0.0, Sum: 0.1]
      [GC Worker Total (ms): Min: 2.8, Avg: 2.9, Max: 2.9, Diff: 0.0, Sum: 11.4]
      [GC Worker End (ms): Min: 1198.7, Avg: 1198.7, Max: 1198.7, Diff: 0.0]
   [Code Root Fixup: 0.0 ms]
   [Code Root Purge: 0.0 ms]
   [Clear CT: 0.2 ms]
   [Other: 0.6 ms]
      [Choose CSet: 0.0 ms]
      [Ref Proc: 0.1 ms]
      [Ref Enq: 0.0 ms]
      [Redirty Cards: 0.4 ms]
      [Humongous Register: 0.0 ms]
      [Humongous Reclaim: 0.0 ms]
      [Free CSet: 0.0 ms]
   [Eden: 2048.0K(2048.0K)->0.0B(1024.0K) Survivors: 0.0B->1024.0K Heap: 3514.3K(6144.0K)->2548.8K(6144.0K)]
 [Times: user=0.00 sys=0.00, real=0.00 secs] 
[GC pause (G1 Evacuation Pause) (young) (initial-mark) (to-space exhausted), 0.0134408 secs]
   [Parallel Time: 8.0 ms, GC Workers: 4]
      [GC Worker Start (ms): Min: 1201.7, Avg: 1201.9, Max: 1202.1, Diff: 0.3]
      [Ext Root Scanning (ms): Min: 0.0, Avg: 3.2, Max: 6.3, Diff: 6.3, Sum: 12.7]
      [Update RS (ms): Min: 0.0, Avg: 0.2, Max: 0.4, Diff: 0.4, Sum: 0.7]
         [Processed Buffers: Min: 0, Avg: 1.5, Max: 3, Diff: 3, Sum: 6]
      [Scan RS (ms): Min: 0.0, Avg: 0.0, Max: 0.0, Diff: 0.0, Sum: 0.0]
      [Code Root Scanning (ms): Min: 0.0, Avg: 0.0, Max: 0.0, Diff: 0.0, Sum: 0.0]
      [Object Copy (ms): Min: 1.4, Avg: 4.4, Max: 7.3, Diff: 5.9, Sum: 17.4]
      [Termination (ms): Min: 0.0, Avg: 0.1, Max: 0.1, Diff: 0.1, Sum: 0.2]
         [Termination Attempts: Min: 1, Avg: 1.0, Max: 1, Diff: 0, Sum: 4]
      [GC Worker Other (ms): Min: 0.0, Avg: 0.0, Max: 0.0, Diff: 0.0, Sum: 0.1]
      [GC Worker Total (ms): Min: 7.6, Avg: 7.8, Max: 7.9, Diff: 0.3, Sum: 31.1]
      [GC Worker End (ms): Min: 1209.7, Avg: 1209.7, Max: 1209.7, Diff: 0.0]
   [Code Root Fixup: 0.0 ms]
   [Code Root Purge: 0.0 ms]
   [Clear CT: 0.1 ms]
   [Other: 5.4 ms]
      [Evacuation Failure: 5.0 ms]
      [Choose CSet: 0.0 ms]
      [Ref Proc: 0.2 ms]
      [Ref Enq: 0.0 ms]
      [Redirty Cards: 0.1 ms]
      [Humongous Register: 0.0 ms]
      [Humongous Reclaim: 0.0 ms]
      [Free CSet: 0.0 ms]
   [Eden: 1024.0K(1024.0K)->0.0B(1024.0K) Survivors: 1024.0K->1024.0K Heap: 3572.8K(6144.0K)->4279.5K(6144.0K)]
 [Times: user=0.03 sys=0.00, real=0.01 secs] 
[GC concurrent-root-region-scan-start]
[GC concurrent-root-region-scan-end, 0.0008090 secs]
[GC concurrent-mark-start]
list size=10000
[GC concurrent-mark-end, 0.0093344 secs]
[GC remark [Finalize Marking, 0.0001600 secs] [GC ref-proc, 0.0000816 secs] [Unloading, 0.0007427 secs], 0.0012200 secs]
 [Times: user=0.00 sys=0.00, real=0.00 secs] 
[GC cleanup 5201K->5201K(6144K), 0.0012848 secs]
 [Times: user=0.00 sys=0.00, real=0.00 secs] 
[GC pause (G1 Evacuation Pause) (young) (to-space exhausted), 0.0289992 secs]
   [Parallel Time: 25.0 ms, GC Workers: 4]
      [GC Worker Start (ms): Min: 1251.7, Avg: 1251.7, Max: 1251.7, Diff: 0.0]
      [Ext Root Scanning (ms): Min: 0.3, Avg: 0.3, Max: 0.3, Diff: 0.0, Sum: 1.2]
      [Update RS (ms): Min: 0.2, Avg: 0.3, Max: 0.4, Diff: 0.2, Sum: 1.0]
         [Processed Buffers: Min: 1, Avg: 1.8, Max: 2, Diff: 1, Sum: 7]
      [Scan RS (ms): Min: 0.0, Avg: 0.0, Max: 0.0, Diff: 0.0, Sum: 0.0]
      [Code Root Scanning (ms): Min: 0.0, Avg: 0.0, Max: 0.0, Diff: 0.0, Sum: 0.0]
      [Object Copy (ms): Min: 24.0, Avg: 24.1, Max: 24.2, Diff: 0.2, Sum: 96.4]
      [Termination (ms): Min: 0.0, Avg: 0.0, Max: 0.1, Diff: 0.1, Sum: 0.1]
         [Termination Attempts: Min: 1, Avg: 1.0, Max: 1, Diff: 0, Sum: 4]
      [GC Worker Other (ms): Min: 0.0, Avg: 0.0, Max: 0.0, Diff: 0.0, Sum: 0.1]
      [GC Worker Total (ms): Min: 24.7, Avg: 24.7, Max: 24.7, Diff: 0.0, Sum: 98.9]
      [GC Worker End (ms): Min: 1276.4, Avg: 1276.4, Max: 1276.4, Diff: 0.0]
   [Code Root Fixup: 0.0 ms]
   [Code Root Purge: 0.0 ms]
   [Clear CT: 0.3 ms]
   [Other: 3.7 ms]
      [Evacuation Failure: 3.3 ms]
      [Choose CSet: 0.0 ms]
      [Ref Proc: 0.3 ms]
      [Ref Enq: 0.0 ms]
      [Redirty Cards: 0.1 ms]
      [Humongous Register: 0.0 ms]
      [Humongous Reclaim: 0.0 ms]
      [Free CSet: 0.0 ms]
   [Eden: 1024.0K(1024.0K)->0.0B(1024.0K) Survivors: 1024.0K->0.0B Heap: 5303.5K(6144.0K)->5303.5K(6144.0K)]
 [Times: user=0.02 sys=0.03, real=0.03 secs] 
[GC pause (G1 Evacuation Pause) (mixed) (to-space exhausted), 0.0041405 secs]
   [Parallel Time: 3.5 ms, GC Workers: 4]
      [GC Worker Start (ms): Min: 1281.5, Avg: 1281.5, Max: 1281.5, Diff: 0.0]
      [Ext Root Scanning (ms): Min: 0.3, Avg: 0.3, Max: 0.3, Diff: 0.0, Sum: 1.2]
      [Update RS (ms): Min: 1.3, Avg: 1.4, Max: 1.5, Diff: 0.2, Sum: 5.7]
         [Processed Buffers: Min: 3, Avg: 3.5, Max: 4, Diff: 1, Sum: 14]
      [Scan RS (ms): Min: 0.0, Avg: 0.0, Max: 0.1, Diff: 0.1, Sum: 0.1]
      [Code Root Scanning (ms): Min: 0.0, Avg: 0.0, Max: 0.0, Diff: 0.0, Sum: 0.0]
      [Object Copy (ms): Min: 1.5, Avg: 1.6, Max: 1.7, Diff: 0.1, Sum: 6.5]
      [Termination (ms): Min: 0.0, Avg: 0.0, Max: 0.1, Diff: 0.1, Sum: 0.1]
         [Termination Attempts: Min: 1, Avg: 1.0, Max: 1, Diff: 0, Sum: 4]
      [GC Worker Other (ms): Min: 0.0, Avg: 0.0, Max: 0.0, Diff: 0.0, Sum: 0.0]
      [GC Worker Total (ms): Min: 3.4, Avg: 3.4, Max: 3.4, Diff: 0.0, Sum: 13.6]
      [GC Worker End (ms): Min: 1284.9, Avg: 1284.9, Max: 1284.9, Diff: 0.0]
   [Code Root Fixup: 0.0 ms]
   [Code Root Purge: 0.0 ms]
   [Clear CT: 0.1 ms]
   [Other: 0.6 ms]
      [Evacuation Failure: 0.4 ms]
      [Choose CSet: 0.0 ms]
      [Ref Proc: 0.1 ms]
      [Ref Enq: 0.0 ms]
      [Redirty Cards: 0.1 ms]
      [Humongous Register: 0.0 ms]
      [Humongous Reclaim: 0.0 ms]
      [Free CSet: 0.0 ms]
   [Eden: 0.0B(1024.0K)->0.0B(1024.0K) Survivors: 0.0B->0.0B Heap: 5303.5K(6144.0K)->5303.5K(6144.0K)]
 [Times: user=0.01 sys=0.00, real=0.00 secs] 
[Full GC (Allocation Failure)  5303K->3610K(6144K), 0.0189011 secs]
   [Eden: 0.0B(1024.0K)->0.0B(1024.0K) Survivors: 0.0B->0.0B Heap: 5303.5K(6144.0K)->3610.0K(6144.0K)], [Metaspace: 2658K->2658K(1056768K)]
 [Times: user=0.06 sys=0.00, real=0.02 secs] 
[GC pause (G1 Evacuation Pause) (young), 0.0015657 secs]
   [Parallel Time: 1.2 ms, GC Workers: 4]
      [GC Worker Start (ms): Min: 1307.2, Avg: 1307.2, Max: 1307.2, Diff: 0.0]
      [Ext Root Scanning (ms): Min: 0.2, Avg: 0.2, Max: 0.2, Diff: 0.0, Sum: 0.8]
      [Update RS (ms): Min: 0.0, Avg: 0.0, Max: 0.2, Diff: 0.2, Sum: 0.2]
         [Processed Buffers: Min: 0, Avg: 0.3, Max: 1, Diff: 1, Sum: 1]
      [Scan RS (ms): Min: 0.0, Avg: 0.0, Max: 0.0, Diff: 0.0, Sum: 0.0]
      [Code Root Scanning (ms): Min: 0.0, Avg: 0.0, Max: 0.0, Diff: 0.0, Sum: 0.0]
      [Object Copy (ms): Min: 0.8, Avg: 0.9, Max: 1.0, Diff: 0.2, Sum: 3.6]
      [Termination (ms): Min: 0.0, Avg: 0.0, Max: 0.0, Diff: 0.0, Sum: 0.0]
         [Termination Attempts: Min: 1, Avg: 1.3, Max: 2, Diff: 1, Sum: 5]
      [GC Worker Other (ms): Min: 0.0, Avg: 0.0, Max: 0.0, Diff: 0.0, Sum: 0.1]
      [GC Worker Total (ms): Min: 1.2, Avg: 1.2, Max: 1.2, Diff: 0.0, Sum: 4.7]
      [GC Worker End (ms): Min: 1308.4, Avg: 1308.4, Max: 1308.4, Diff: 0.0]
   [Code Root Fixup: 0.0 ms]
   [Code Root Purge: 0.0 ms]
   [Clear CT: 0.1 ms]
   [Other: 0.2 ms]
      [Choose CSet: 0.0 ms]
      [Ref Proc: 0.1 ms]
      [Ref Enq: 0.0 ms]
      [Redirty Cards: 0.0 ms]
      [Humongous Register: 0.0 ms]
      [Humongous Reclaim: 0.0 ms]
      [Free CSet: 0.0 ms]
   [Eden: 1024.0K(1024.0K)->0.0B(1024.0K) Survivors: 0.0B->1024.0K Heap: 4634.0K(6144.0K)->4209.2K(6144.0K)]
 [Times: user=0.00 sys=0.00, real=0.00 secs] 
[GC pause (G1 Evacuation Pause) (young) (initial-mark) (to-space exhausted), 0.0197143 secs]
   [Parallel Time: 16.6 ms, GC Workers: 4]
      [GC Worker Start (ms): Min: 1310.1, Avg: 1310.2, Max: 1310.4, Diff: 0.3]
      [Ext Root Scanning (ms): Min: 0.0, Avg: 0.3, Max: 0.4, Diff: 0.4, Sum: 1.0]
      [Update RS (ms): Min: 0.0, Avg: 0.1, Max: 0.3, Diff: 0.3, Sum: 0.4]
         [Processed Buffers: Min: 0, Avg: 1.5, Max: 3, Diff: 3, Sum: 6]
      [Scan RS (ms): Min: 0.0, Avg: 0.0, Max: 0.0, Diff: 0.0, Sum: 0.0]
      [Code Root Scanning (ms): Min: 0.0, Avg: 0.0, Max: 0.0, Diff: 0.0, Sum: 0.0]
      [Object Copy (ms): Min: 15.8, Avg: 15.9, Max: 16.0, Diff: 0.1, Sum: 63.6]
      [Termination (ms): Min: 0.0, Avg: 0.0, Max: 0.0, Diff: 0.0, Sum: 0.1]
         [Termination Attempts: Min: 1, Avg: 1.3, Max: 2, Diff: 1, Sum: 5]
      [GC Worker Other (ms): Min: 0.0, Avg: 0.0, Max: 0.0, Diff: 0.0, Sum: 0.0]
      [GC Worker Total (ms): Min: 16.1, Avg: 16.3, Max: 16.4, Diff: 0.3, Sum: 65.2]
      [GC Worker End (ms): Min: 1326.5, Avg: 1326.5, Max: 1326.5, Diff: 0.0]
   [Code Root Fixup: 0.0 ms]
   [Code Root Purge: 0.0 ms]
   [Clear CT: 0.2 ms]
   [Other: 2.9 ms]
      [Evacuation Failure: 2.5 ms]
      [Choose CSet: 0.0 ms]
      [Ref Proc: 0.2 ms]
      [Ref Enq: 0.0 ms]
      [Redirty Cards: 0.1 ms]
      [Humongous Register: 0.0 ms]
      [Humongous Reclaim: 0.0 ms]
      [Free CSet: 0.0 ms]
   [Eden: 1024.0K(1024.0K)->0.0B(1024.0K) Survivors: 1024.0K->0.0B Heap: 5233.2K(6144.0K)->5233.2K(6144.0K)]
 [Times: user=0.00 sys=0.01, real=0.02 secs] 
[GC concurrent-root-region-scan-start]
[GC concurrent-root-region-scan-end, 0.0000140 secs]
[GC concurrent-mark-start]
[GC pause (G1 Evacuation Pause) (young), 0.0016455 secs]
   [Parallel Time: 1.4 ms, GC Workers: 4]
      [GC Worker Start (ms): Min: 1330.9, Avg: 1331.3, Max: 1332.3, Diff: 1.3]
      [Ext Root Scanning (ms): Min: 0.0, Avg: 0.2, Max: 0.2, Diff: 0.2, Sum: 0.7]
      [Update RS (ms): Min: 0.0, Avg: 0.8, Max: 1.0, Diff: 1.0, Sum: 3.0]
         [Processed Buffers: Min: 0, Avg: 3.0, Max: 4, Diff: 4, Sum: 12]
      [Scan RS (ms): Min: 0.0, Avg: 0.0, Max: 0.0, Diff: 0.0, Sum: 0.0]
      [Code Root Scanning (ms): Min: 0.0, Avg: 0.0, Max: 0.0, Diff: 0.0, Sum: 0.0]
      [Object Copy (ms): Min: 0.0, Avg: 0.0, Max: 0.0, Diff: 0.0, Sum: 0.0]
      [Termination (ms): Min: 0.0, Avg: 0.1, Max: 0.1, Diff: 0.1, Sum: 0.3]
         [Termination Attempts: Min: 1, Avg: 1.0, Max: 1, Diff: 0, Sum: 4]
      [GC Worker Other (ms): Min: 0.0, Avg: 0.0, Max: 0.0, Diff: 0.0, Sum: 0.0]
      [GC Worker Total (ms): Min: 0.0, Avg: 1.0, Max: 1.4, Diff: 1.3, Sum: 4.1]
      [GC Worker End (ms): Min: 1332.3, Avg: 1332.3, Max: 1332.3, Diff: 0.0]
   [Code Root Fixup: 0.0 ms]
   [Code Root Purge: 0.0 ms]
   [Clear CT: 0.0 ms]
   [Other: 0.2 ms]
      [Choose CSet: 0.0 ms]
      [Ref Proc: 0.1 ms]
      [Ref Enq: 0.0 ms]
      [Redirty Cards: 0.0 ms]
      [Humongous Register: 0.0 ms]
      [Humongous Reclaim: 0.0 ms]
      [Free CSet: 0.0 ms]
   [Eden: 0.0B(1024.0K)->0.0B(1024.0K) Survivors: 0.0B->0.0B Heap: 5233.2K(6144.0K)->5233.2K(6144.0K)]
 [Times: user=0.00 sys=0.02, real=0.00 secs] 
[Full GC (Allocation Failure)  5233K->4715K(6144K), 0.0143916 secs]
   [Eden: 0.0B(1024.0K)->0.0B(1024.0K) Survivors: 0.0B->0.0B Heap: 5233.2K(6144.0K)->4715.6K(6144.0K)], [Metaspace: 2658K->2658K(1056768K)]
 [Times: user=0.00 sys=0.00, real=0.01 secs] 
[GC concurrent-mark-abort]
[GC pause (G1 Evacuation Pause) (young) (to-space exhausted), 0.0090503 secs]
   [Parallel Time: 6.2 ms, GC Workers: 4]
      [GC Worker Start (ms): Min: 1348.7, Avg: 1348.7, Max: 1348.7, Diff: 0.0]
      [Ext Root Scanning (ms): Min: 0.2, Avg: 0.2, Max: 0.2, Diff: 0.0, Sum: 0.8]
      [Update RS (ms): Min: 0.0, Avg: 0.0, Max: 0.0, Diff: 0.0, Sum: 0.0]
         [Processed Buffers: Min: 0, Avg: 0.3, Max: 1, Diff: 1, Sum: 1]
      [Scan RS (ms): Min: 0.0, Avg: 0.0, Max: 0.0, Diff: 0.0, Sum: 0.0]
      [Code Root Scanning (ms): Min: 0.0, Avg: 0.0, Max: 0.0, Diff: 0.0, Sum: 0.0]
      [Object Copy (ms): Min: 5.9, Avg: 5.9, Max: 5.9, Diff: 0.1, Sum: 23.6]
      [Termination (ms): Min: 0.0, Avg: 0.0, Max: 0.0, Diff: 0.0, Sum: 0.1]
         [Termination Attempts: Min: 1, Avg: 1.8, Max: 3, Diff: 2, Sum: 7]
      [GC Worker Other (ms): Min: 0.0, Avg: 0.0, Max: 0.0, Diff: 0.0, Sum: 0.0]
      [GC Worker Total (ms): Min: 6.1, Avg: 6.1, Max: 6.2, Diff: 0.0, Sum: 24.6]
      [GC Worker End (ms): Min: 1354.9, Avg: 1354.9, Max: 1354.9, Diff: 0.0]
   [Code Root Fixup: 0.0 ms]
   [Code Root Purge: 0.0 ms]
   [Clear CT: 0.0 ms]
   [Other: 2.8 ms]
      [Evacuation Failure: 2.2 ms]
      [Choose CSet: 0.0 ms]
      [Ref Proc: 0.1 ms]
      [Ref Enq: 0.0 ms]
      [Redirty Cards: 0.4 ms]
      [Humongous Register: 0.0 ms]
      [Humongous Reclaim: 0.0 ms]
      [Free CSet: 0.0 ms]
   [Eden: 1024.0K(1024.0K)->0.0B(1024.0K) Survivors: 0.0B->0.0B Heap: 5739.6K(6144.0K)->5739.6K(6144.0K)]
 [Times: user=0.00 sys=0.00, real=0.01 secs] 
[GC pause (G1 Evacuation Pause) (young) (initial-mark), 0.0019007 secs]
   [Parallel Time: 1.1 ms, GC Workers: 4]
      [GC Worker Start (ms): Min: 1358.1, Avg: 1358.1, Max: 1358.1, Diff: 0.0]
      [Ext Root Scanning (ms): Min: 0.2, Avg: 0.2, Max: 0.2, Diff: 0.0, Sum: 0.8]
      [Update RS (ms): Min: 0.6, Avg: 0.7, Max: 0.8, Diff: 0.2, Sum: 2.7]
         [Processed Buffers: Min: 1, Avg: 3.0, Max: 4, Diff: 3, Sum: 12]
      [Scan RS (ms): Min: 0.0, Avg: 0.0, Max: 0.0, Diff: 0.0, Sum: 0.0]
      [Code Root Scanning (ms): Min: 0.0, Avg: 0.0, Max: 0.0, Diff: 0.0, Sum: 0.0]
      [Object Copy (ms): Min: 0.0, Avg: 0.0, Max: 0.0, Diff: 0.0, Sum: 0.0]
      [Termination (ms): Min: 0.0, Avg: 0.1, Max: 0.2, Diff: 0.2, Sum: 0.5]
         [Termination Attempts: Min: 1, Avg: 1.0, Max: 1, Diff: 0, Sum: 4]
      [GC Worker Other (ms): Min: 0.0, Avg: 0.0, Max: 0.0, Diff: 0.0, Sum: 0.0]
      [GC Worker Total (ms): Min: 1.0, Avg: 1.0, Max: 1.0, Diff: 0.0, Sum: 4.1]
      [GC Worker End (ms): Min: 1359.1, Avg: 1359.1, Max: 1359.1, Diff: 0.0]
   [Code Root Fixup: 0.0 ms]
   [Code Root Purge: 0.0 ms]
   [Clear CT: 0.3 ms]
   [Other: 0.5 ms]
      [Choose CSet: 0.0 ms]
      [Ref Proc: 0.1 ms]
      [Ref Enq: 0.0 ms]
      [Redirty Cards: 0.4 ms]
      [Humongous Register: 0.0 ms]
      [Humongous Reclaim: 0.0 ms]
      [Free CSet: 0.0 ms]
   [Eden: 0.0B(1024.0K)->0.0B(1024.0K) Survivors: 0.0B->0.0B Heap: 5739.6K(6144.0K)->5739.6K(6144.0K)]
 [Times: user=0.00 sys=0.00, real=0.00 secs] 
[GC concurrent-root-region-scan-start]
[GC concurrent-root-region-scan-end, 0.0000093 secs]
[GC concurrent-mark-start]
[Full GC (Allocation Failure)  5739K->5232K(6144K), 0.0166473 secs]
   [Eden: 0.0B(1024.0K)->0.0B(1024.0K) Survivors: 0.0B->0.0B Heap: 5739.6K(6144.0K)->5232.2K(6144.0K)], [Metaspace: 2658K->2658K(1056768K)]
 [Times: user=0.01 sys=0.00, real=0.02 secs] 
[Full GC (Allocation Failure)  5232K->5232K(6144K), 0.0142316 secs]
   [Eden: 0.0B(1024.0K)->0.0B(1024.0K) Survivors: 0.0B->0.0B Heap: 5232.2K(6144.0K)->5232.2K(6144.0K)], [Metaspace: 2658K->2658K(1056768K)]
 [Times: user=0.02 sys=0.00, real=0.01 secs] 
[GC concurrent-mark-abort]
[GC pause (G1 Evacuation Pause) (young), 0.0004880 secs]
   [Parallel Time: 0.3 ms, GC Workers: 4]
      [GC Worker Start (ms): Min: 1391.6, Avg: 1391.6, Max: 1391.6, Diff: 0.0]
      [Ext Root Scanning (ms): Min: 0.2, Avg: 0.2, Max: 0.2, Diff: 0.0, Sum: 0.8]
      [Update RS (ms): Min: 0.0, Avg: 0.0, Max: 0.0, Diff: 0.0, Sum: 0.0]
         [Processed Buffers: Min: 0, Avg: 0.3, Max: 1, Diff: 1, Sum: 1]
      [Scan RS (ms): Min: 0.0, Avg: 0.0, Max: 0.0, Diff: 0.0, Sum: 0.0]
      [Code Root Scanning (ms): Min: 0.0, Avg: 0.0, Max: 0.0, Diff: 0.0, Sum: 0.0]
      [Object Copy (ms): Min: 0.0, Avg: 0.0, Max: 0.0, Diff: 0.0, Sum: 0.0]
      [Termination (ms): Min: 0.0, Avg: 0.0, Max: 0.0, Diff: 0.0, Sum: 0.0]
         [Termination Attempts: Min: 1, Avg: 1.0, Max: 1, Diff: 0, Sum: 4]
      [GC Worker Other (ms): Min: 0.0, Avg: 0.0, Max: 0.0, Diff: 0.0, Sum: 0.0]
      [GC Worker Total (ms): Min: 0.2, Avg: 0.2, Max: 0.2, Diff: 0.0, Sum: 0.9]
      [GC Worker End (ms): Min: 1391.9, Avg: 1391.9, Max: 1391.9, Diff: 0.0]
   [Code Root Fixup: 0.0 ms]
   [Code Root Purge: 0.0 ms]
   [Clear CT: 0.1 ms]
   [Other: 0.1 ms]
      [Choose CSet: 0.0 ms]
      [Ref Proc: 0.1 ms]
      [Ref Enq: 0.0 ms]
      [Redirty Cards: 0.0 ms]
      [Humongous Register: 0.0 ms]
      [Humongous Reclaim: 0.0 ms]
      [Free CSet: 0.0 ms]
   [Eden: 0.0B(1024.0K)->0.0B(1024.0K) Survivors: 0.0B->0.0B Heap: 5232.2K(6144.0K)->5232.2K(6144.0K)]
 [Times: user=0.00 sys=0.00, real=0.00 secs] 
[GC pause (G1 Evacuation Pause) (young) (initial-mark), 0.0004175 secs]
   [Parallel Time: 0.2 ms, GC Workers: 4]
      [GC Worker Start (ms): Min: 1392.5, Avg: 1392.5, Max: 1392.5, Diff: 0.0]
      [Ext Root Scanning (ms): Min: 0.2, Avg: 0.2, Max: 0.2, Diff: 0.0, Sum: 0.8]
      [Update RS (ms): Min: 0.0, Avg: 0.0, Max: 0.0, Diff: 0.0, Sum: 0.0]
         [Processed Buffers: Min: 0, Avg: 0.3, Max: 1, Diff: 1, Sum: 1]
      [Scan RS (ms): Min: 0.0, Avg: 0.0, Max: 0.0, Diff: 0.0, Sum: 0.0]
      [Code Root Scanning (ms): Min: 0.0, Avg: 0.0, Max: 0.0, Diff: 0.0, Sum: 0.0]
      [Object Copy (ms): Min: 0.0, Avg: 0.0, Max: 0.0, Diff: 0.0, Sum: 0.0]
      [Termination (ms): Min: 0.0, Avg: 0.0, Max: 0.0, Diff: 0.0, Sum: 0.0]
         [Termination Attempts: Min: 1, Avg: 1.0, Max: 1, Diff: 0, Sum: 4]
      [GC Worker Other (ms): Min: 0.0, Avg: 0.0, Max: 0.0, Diff: 0.0, Sum: 0.0]
      [GC Worker Total (ms): Min: 0.2, Avg: 0.2, Max: 0.2, Diff: 0.0, Sum: 0.9]
      [GC Worker End (ms): Min: 1392.8, Avg: 1392.8, Max: 1392.8, Diff: 0.0]
   [Code Root Fixup: 0.0 ms]
   [Code Root Purge: 0.0 ms]
   [Clear CT: 0.0 ms]
   [Other: 0.1 ms]
      [Choose CSet: 0.0 ms]
      [Ref Proc: 0.1 ms]
      [Ref Enq: 0.0 ms]
      [Redirty Cards: 0.0 ms]
      [Humongous Register: 0.0 ms]
      [Humongous Reclaim: 0.0 ms]
      [Free CSet: 0.0 ms]
   [Eden: 0.0B(1024.0K)->0.0B(1024.0K) Survivors: 0.0B->0.0B Heap: 5232.2K(6144.0K)->5232.2K(6144.0K)]
 [Times: user=0.00 sys=0.00, real=0.00 secs] 
[GC concurrent-root-region-scan-start]
[GC concurrent-root-region-scan-end, 0.0000098 secs]
[GC concurrent-mark-start]
[Full GC (Allocation Failure)  5232K->508K(6144K), 0.0049858 secs]
   [Eden: 0.0B(1024.0K)->0.0B(3072.0K) Survivors: 0.0B->0.0B Heap: 5232.2K(6144.0K)->508.2K(6144.0K)], [Metaspace: 2659K->2659K(1056768K)]
 [Times: user=0.01 sys=0.00, real=0.00 secs] 
[GC concurrent-mark-abort]
Exception in thread "main" java.lang.OutOfMemoryError: Java heap space
	at java.util.Arrays.copyOfRange(Arrays.java:3664)
	at java.lang.String.<init>(String.java:207)
	at java.lang.StringBuilder.toString(StringBuilder.java:407)
	at gc.SimpleTest.main(SimpleTest.java:22)
Heap
 garbage-first heap   total 6144K, used 508K [0x00000000ffa00000, 0x00000000ffb00030, 0x0000000100000000)
  region size 1024K, 1 young (1024K), 0 survivors (0K)
 Metaspace       used 2689K, capacity 4486K, committed 4864K, reserved 1056768K
  class space    used 285K, capacity 386K, committed 512K, reserved 1048576K

```

简析如下

```java
[GC pause (G1 Evacuation Pause) (young), 0.0044882 secs]
   [Parallel Time: 3.7 ms, GC Workers: 3]
      [GC Worker Start (ms): Min: 14763.7, Avg: 14763.8, Max: 14763.8,
Diff: 0.1]
      #扫描根节点
      [Ext Root Scanning (ms): Min: 0.2, Avg: 0.3, Max: 0.3, Diff: 0.1,
Sum: 0.8]
      #更新RS区域所消耗的时间
      [Update RS (ms): Min: 1.8, Avg: 1.9, Max: 1.9, Diff: 0.2, Sum: 5.6]
         [Processed Buffers: Min: 1, Avg: 1.7, Max: 3, Diff: 2, Sum: 5]
      [Scan RS (ms): Min: 0.0, Avg: 0.0, Max: 0.0, Diff: 0.0, Sum: 0.0]
      [Code Root Scanning (ms): Min: 0.0, Avg: 0.0, Max: 0.0, Diff: 0.0,
Sum: 0.0]
      #对象拷贝
      [Object Copy (ms): Min: 1.1, Avg: 1.2, Max: 1.3, Diff: 0.2, Sum:
3.6]
      [Termination (ms): Min: 0.0, Avg: 0.1, Max: 0.2, Diff: 0.2, Sum:
0.2]
         [Termination Attempts: Min: 1, Avg: 1.0, Max: 1, Diff: 0, Sum:
3]
      [GC Worker Other (ms): Min: 0.0, Avg: 0.0, Max: 0.0, Diff: 0.0,
Sum: 0.0]
      [GC Worker Total (ms): Min: 3.4, Avg: 3.4, Max: 3.5, Diff: 0.1,
Sum: 10.3]
      [GC Worker End (ms): Min: 14767.2, Avg: 14767.2, Max: 14767.3,
Diff: 0.1]
   [Code Root Fixup: 0.0 ms]
   [Code Root Purge: 0.0 ms]
   [Clear CT: 0.0 ms] #清空CardTable
   [Other: 0.7 ms]
      [Choose CSet: 0.0 ms] #选取CSet
      [Ref Proc: 0.5 ms] #弱引用、软引用的处理耗时
      [Ref Enq: 0.0 ms] #弱引用、软引用的入队耗时
      [Redirty Cards: 0.0 ms]
      [Humongous Register: 0.0 ms] #大对象区域注册耗时
      [Humongous Reclaim: 0.0 ms] #大对象区域回收耗时
      [Free CSet: 0.0 ms]
   [Eden: 7168.0K(7168.0K)‐>0.0B(13.0M) Survivors: 2048.0K‐>2048.0K Heap:
55.5M(192.0M)‐>48.5M(192.0M)] #年轻代的大小统计
   [Times: user=0.00 sys=0.00, real=0.00 secs]
```

